# Part 015: OOP พื้นฐาน (Object-Oriented Programming)
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- สร้างและใช้งาน Class ได้
- เข้าใจ \_\_init\_\_ และ self ได้
- ใช้ Instance, Class, Static Variables ได้
- สร้าง Instance, Class, Static Methods ได้
- ใช้ Properties ได้
- เขียน Magic Methods (\_\_str\_\_, \_\_repr\_\_, \_\_add\_\_, \_\_len\_\_, \_\_eq\_\_) ได้

---

## 1. Class พื้นฐาน

```python
# Class = แบบแผน (blueprint) สำหรับสร้าง objects
# Object = instance ของ class

# สร้าง class อย่างง่าย
class Dog:
    """แทนสุนัข"""
    
    # Class variable - ใช้ร่วมกันทุก instance
    species = "Canis lupus familiaris"
    count = 0
    
    # __init__ - constructor (เรียกเมื่อสร้าง object)
    def __init__(self, name: str, breed: str, age: int):
        """สร้าง Dog instance"""
        # Instance variables - แต่ละ instance มีเป็นของตัวเอง
        self.name = name
        self.breed = breed
        self.age = age
        self._tricks = []  # _ = private by convention
        
        Dog.count += 1  # เพิ่ม counter
    
    # Instance method - ทำงานกับ instance
    def bark(self) -> str:
        return f"{self.name}: โฮ่ง โฮ่ง!"
    
    def learn_trick(self, trick: str) -> None:
        self._tricks.append(trick)
        print(f"{self.name} เรียนรู้ trick ใหม่: {trick}")
    
    def show_tricks(self) -> None:
        if self._tricks:
            print(f"Tricks ของ {self.name}: {', '.join(self._tricks)}")
        else:
            print(f"{self.name} ยังไม่รู้ trick ใด")
    
    def birthday(self) -> None:
        self.age += 1
        print(f"Happy Birthday {self.name}! อายุ {self.age} ปีแล้ว")
    
    # Class method - ทำงานกับ class (ไม่ใช่ instance)
    @classmethod
    def get_count(cls) -> int:
        return cls.count
    
    @classmethod
    def create_puppy(cls, name: str, breed: str) -> "Dog":
        """Factory method สร้าง puppy"""
        return cls(name, breed, age=0)
    
    # Static method - ไม่เกี่ยวกับ instance หรือ class
    @staticmethod
    def is_valid_age(age: int) -> bool:
        return 0 <= age <= 25
    
    # __str__ - Human-readable string
    def __str__(self) -> str:
        return f"Dog({self.name}, {self.breed}, อายุ {self.age} ปี)"
    
    # __repr__ - Developer string (unambiguous)
    def __repr__(self) -> str:
        return f"Dog(name={self.name!r}, breed={self.breed!r}, age={self.age})"

# สร้าง instances
dog1 = Dog("Max", "Golden Retriever", 3)
dog2 = Dog("Bella", "Bulldog", 5)
puppy = Dog.create_puppy("Charlie", "Poodle")

# ใช้งาน
print(dog1.bark())
print(dog2.bark())

dog1.learn_trick("นั่ง")
dog1.learn_trick("ยืน")
dog1.show_tricks()
dog2.show_tricks()

dog1.birthday()

print(f"\nClass variable: {Dog.species}")
print(f"จำนวน dogs: {Dog.get_count()}")
print(f"Valid age 5: {Dog.is_valid_age(5)}")
print(f"Valid age 30: {Dog.is_valid_age(30)}")

print(f"\nstr: {dog1}")
print(f"repr: {repr(dog1)}")

# Instance variables
print(f"\ndog1.name: {dog1.name}")
print(f"dog2.name: {dog2.name}")

# Class variables
print(f"Dog.species: {Dog.species}")
print(f"dog1.species: {dog1.species}")  # inherit จาก class

# แก้ class variable ที่ instance level
dog1.species = "Canis lupus familiaris (modified)"  # สร้าง instance variable
print(f"dog1.species: {dog1.species}")  # instance variable
print(f"Dog.species: {Dog.species}")    # class variable ไม่เปลี่ยน
print(f"dog2.species: {dog2.species}")  # ยังใช้ class variable
```

---

## 2. Instance, Class, Static Variables

```python
class Counter:
    # Class variable
    total_instances = 0    # shared across all instances
    _registry = {}         # class-level registry
    
    def __init__(self, name: str, start: int = 0):
        # Instance variables
        self.name = name    # public
        self._value = start  # protected (convention)
        self.__secret = "hidden"  # private (name mangled)
        
        Counter.total_instances += 1
        Counter._registry[name] = self
    
    @property
    def value(self):
        return self._value
    
    def increment(self, by: int = 1):
        self._value += by
    
    @classmethod
    def get_all(cls):
        return dict(cls._registry)
    
    @classmethod
    def reset_all(cls):
        for counter in cls._registry.values():
            counter._value = 0
    
    @staticmethod
    def validate_increment(n: int) -> bool:
        return isinstance(n, int) and n > 0
    
    def __del__(self):
        Counter.total_instances -= 1
        Counter._registry.pop(self.name, None)

# ทดสอบ
c1 = Counter("visits", 100)
c2 = Counter("errors", 0)
c3 = Counter("warnings", 5)

print(f"Total: {Counter.total_instances}")  # 3

c1.increment(10)
c2.increment()
c3.increment(3)

print(f"visits: {c1.value}")    # 110
print(f"errors: {c2.value}")    # 1
print(f"warnings: {c3.value}")  # 8

# ดู name mangling
print(f"Public: {c1.name}")
print(f"Protected: {c1._value}")
# c1.__secret  # AttributeError!
print(f"Mangled: {c1._Counter__secret}")  # _ClassName__attr

# Class registry
print("\nAll counters:")
for name, counter in Counter.get_all().items():
    print(f"  {name}: {counter.value}")

Counter.reset_all()
print("\nAfter reset:")
for name, counter in Counter.get_all().items():
    print(f"  {name}: {counter.value}")
```

---

## 3. Properties (Getters/Setters)

```python
class Temperature:
    """จัดการอุณหภูมิพร้อม validation"""
    
    def __init__(self, celsius: float = 0.0):
        self._celsius = celsius  # internal storage
    
    @property
    def celsius(self) -> float:
        """Getter สำหรับ celsius"""
        return self._celsius
    
    @celsius.setter
    def celsius(self, value: float) -> None:
        """Setter สำหรับ celsius พร้อม validation"""
        if not isinstance(value, (int, float)):
            raise TypeError("อุณหภูมิต้องเป็นตัวเลข")
        if value < -273.15:
            raise ValueError("อุณหภูมิต่ำกว่า Absolute Zero ไม่ได้")
        self._celsius = float(value)
    
    @celsius.deleter
    def celsius(self) -> None:
        print("Deleting temperature...")
        self._celsius = 0.0
    
    @property
    def fahrenheit(self) -> float:
        """แปลงเป็น Fahrenheit (computed property)"""
        return self._celsius * 9/5 + 32
    
    @fahrenheit.setter
    def fahrenheit(self, value: float) -> None:
        self.celsius = (value - 32) * 5/9
    
    @property
    def kelvin(self) -> float:
        """แปลงเป็น Kelvin (read-only)"""
        return self._celsius + 273.15
    
    def __str__(self) -> str:
        return f"{self.celsius:.1f}°C / {self.fahrenheit:.1f}°F / {self.kelvin:.2f}K"

# ทดสอบ
t = Temperature(25.0)
print(t)  # 25.0°C / 77.0°F / 298.15K

t.celsius = 100.0
print(t)  # 100.0°C / 212.0°F / 373.15K

t.fahrenheit = 32.0  # 0°C
print(t)  # 0.0°C / 32.0°F / 273.15K

# Validation
try:
    t.celsius = -300   # ต่ำกว่า absolute zero
except ValueError as e:
    print(f"Error: {e}")

try:
    t.celsius = "hot"  # ไม่ใช่ตัวเลข
except TypeError as e:
    print(f"Error: {e}")

# Read-only property
try:
    t.kelvin = 300  # kelvin ไม่มี setter
except AttributeError as e:
    print(f"Error: {e}")

# Delete property
del t.celsius
print(f"หลัง delete: {t.celsius}°C")

# ตัวอย่างเพิ่มเติม: Person class
class Person:
    def __init__(self, first_name: str, last_name: str, age: int):
        self.first_name = first_name
        self.last_name = last_name
        self.age = age
    
    @property
    def full_name(self) -> str:
        return f"{self.first_name} {self.last_name}"
    
    @full_name.setter
    def full_name(self, name: str) -> None:
        parts = name.strip().split(None, 1)
        self.first_name = parts[0]
        self.last_name = parts[1] if len(parts) > 1 else ""
    
    @property
    def age(self) -> int:
        return self._age
    
    @age.setter
    def age(self, value: int) -> None:
        if not isinstance(value, int) or value < 0:
            raise ValueError("อายุต้องเป็น int ที่ไม่ติดลบ")
        self._age = value
    
    def __repr__(self) -> str:
        return f"Person(full_name={self.full_name!r}, age={self.age})"

p = Person("Alice", "Smith", 30)
print(p.full_name)   # Alice Smith

p.full_name = "Bob Johnson"
print(p.first_name)  # Bob
print(p.last_name)   # Johnson

try:
    p.age = -5
except ValueError as e:
    print(f"Error: {e}")
```

---

## 4. Magic Methods (Dunder Methods)

### 4.1 \_\_str\_\_ และ \_\_repr\_\_

```python
class Point:
    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y
    
    def __str__(self) -> str:
        """Human-readable - ใช้โดย print() และ str()"""
        return f"({self.x}, {self.y})"
    
    def __repr__(self) -> str:
        """Developer-readable - ใช้โดย repr() และ debug"""
        return f"Point(x={self.x}, y={self.y})"
    
    def __format__(self, spec: str) -> str:
        """ใช้โดย f-string และ format()"""
        if spec == "polar":
            import math
            r = math.sqrt(self.x**2 + self.y**2)
            theta = math.degrees(math.atan2(self.y, self.x))
            return f"({r:.2f}∠{theta:.1f}°)"
        return self.__str__()

p = Point(3, 4)
print(str(p))          # (3, 4)
print(repr(p))         # Point(x=3, y=4)
print(p)               # (3, 4)
print(f"Point: {p}")           # Point: (3, 4)
print(f"Polar: {p:polar}")     # Polar: (5.00∠53.1°)

# List of Points ใช้ repr
points = [Point(1, 2), Point(3, 4)]
print(points)  # [Point(x=1, y=2), Point(x=3, y=4)]  ← ใช้ repr
```

### 4.2 Arithmetic Magic Methods

```python
import math

class Vector:
    """2D Vector"""
    
    def __init__(self, x: float, y: float):
        self.x = float(x)
        self.y = float(y)
    
    def __add__(self, other: "Vector") -> "Vector":
        """v1 + v2"""
        if isinstance(other, Vector):
            return Vector(self.x + other.x, self.y + other.y)
        return NotImplemented
    
    def __sub__(self, other: "Vector") -> "Vector":
        """v1 - v2"""
        if isinstance(other, Vector):
            return Vector(self.x - other.x, self.y - other.y)
        return NotImplemented
    
    def __mul__(self, scalar: float) -> "Vector":
        """v * scalar"""
        if isinstance(scalar, (int, float)):
            return Vector(self.x * scalar, self.y * scalar)
        return NotImplemented
    
    def __rmul__(self, scalar: float) -> "Vector":
        """scalar * v"""
        return self.__mul__(scalar)
    
    def __truediv__(self, scalar: float) -> "Vector":
        """v / scalar"""
        if scalar == 0:
            raise ZeroDivisionError("หารด้วยศูนย์ไม่ได้")
        return Vector(self.x / scalar, self.y / scalar)
    
    def __neg__(self) -> "Vector":
        """-v"""
        return Vector(-self.x, -self.y)
    
    def __abs__(self) -> float:
        """abs(v) - magnitude"""
        return math.sqrt(self.x**2 + self.y**2)
    
    def __eq__(self, other: object) -> bool:
        """v1 == v2"""
        if isinstance(other, Vector):
            return math.isclose(self.x, other.x) and math.isclose(self.y, other.y)
        return NotImplemented
    
    def __bool__(self) -> bool:
        """bool(v) - False ถ้า zero vector"""
        return abs(self) != 0
    
    def dot(self, other: "Vector") -> float:
        """Dot product"""
        return self.x * other.x + self.y * other.y
    
    def normalize(self) -> "Vector":
        """Unit vector"""
        mag = abs(self)
        if mag == 0:
            raise ValueError("Zero vector ไม่สามารถ normalize ได้")
        return self / mag
    
    def __str__(self) -> str:
        return f"({self.x:.2f}, {self.y:.2f})"
    
    def __repr__(self) -> str:
        return f"Vector({self.x}, {self.y})"

# ทดสอบ
v1 = Vector(3, 4)
v2 = Vector(1, 2)

print(f"v1 = {v1}")
print(f"v2 = {v2}")
print(f"v1 + v2 = {v1 + v2}")
print(f"v1 - v2 = {v1 - v2}")
print(f"v1 * 2 = {v1 * 2}")
print(f"3 * v1 = {3 * v1}")
print(f"v1 / 2 = {v1 / 2}")
print(f"-v1 = {-v1}")
print(f"|v1| = {abs(v1):.2f}")
print(f"v1 · v2 = {v1.dot(v2):.2f}")
print(f"v1 normalize = {v1.normalize()}")
print(f"v1 == Vector(3, 4): {v1 == Vector(3, 4)}")
print(f"bool(v1): {bool(v1)}")
print(f"bool(Vector(0,0)): {bool(Vector(0, 0))}")
```

### 4.3 Comparison Magic Methods

```python
from functools import total_ordering

@total_ordering  # ถ้ามี __eq__ และ 1 comparison method - generate ที่เหลือ
class Student:
    def __init__(self, name: str, gpa: float):
        self.name = name
        self.gpa = gpa
    
    def __eq__(self, other: object) -> bool:
        if isinstance(other, Student):
            return self.gpa == other.gpa
        return NotImplemented
    
    def __lt__(self, other: "Student") -> bool:
        if isinstance(other, Student):
            return self.gpa < other.gpa
        return NotImplemented
    
    # @total_ordering จะ generate __le__, __gt__, __ge__
    
    def __str__(self) -> str:
        return f"{self.name} (GPA: {self.gpa:.2f})"
    
    def __repr__(self) -> str:
        return f"Student({self.name!r}, {self.gpa})"
    
    def __hash__(self) -> int:
        """ต้องมีถ้ามี __eq__"""
        return hash((self.name, self.gpa))

students = [
    Student("Charlie", 3.5),
    Student("Alice", 3.8),
    Student("Bob", 3.2),
    Student("Diana", 3.9),
]

print("Sorted:")
for s in sorted(students):
    print(f"  {s}")

print(f"\nBest: {max(students)}")
print(f"Worst: {min(students)}")

alice = Student("Alice", 3.8)
charlie = Student("Charlie", 3.5)

print(f"\nalice > charlie: {alice > charlie}")
print(f"alice <= alice: {alice <= alice}")
print(f"charlie < alice: {charlie < alice}")

# Set (ต้องใช้ __hash__)
unique = {Student("A", 3.5), Student("B", 3.5), Student("A", 3.5)}
print(f"Unique students: {len(unique)}")
```

### 4.4 Container Magic Methods

```python
class Stack:
    """Stack data structure"""
    
    def __init__(self, *items):
        self._data = list(items)
    
    def push(self, item) -> None:
        self._data.append(item)
    
    def pop(self):
        if not self._data:
            raise IndexError("Stack ว่าง")
        return self._data.pop()
    
    def peek(self):
        if not self._data:
            raise IndexError("Stack ว่าง")
        return self._data[-1]
    
    # Container methods
    def __len__(self) -> int:
        """len(stack)"""
        return len(self._data)
    
    def __getitem__(self, index):
        """stack[i]"""
        return self._data[index]
    
    def __setitem__(self, index, value) -> None:
        """stack[i] = value"""
        self._data[index] = value
    
    def __delitem__(self, index) -> None:
        """del stack[i]"""
        del self._data[index]
    
    def __contains__(self, item) -> bool:
        """item in stack"""
        return item in self._data
    
    def __iter__(self):
        """for item in stack"""
        return iter(self._data)
    
    def __reversed__(self):
        """reversed(stack)"""
        return reversed(self._data)
    
    def __bool__(self) -> bool:
        """bool(stack) - False ถ้าว่าง"""
        return bool(self._data)
    
    def __str__(self) -> str:
        return f"Stack({self._data})"
    
    def __repr__(self) -> str:
        return f"Stack(*{self._data!r})"

# ทดสอบ
s = Stack(1, 2, 3)
s.push(4)
s.push(5)

print(f"Stack: {s}")
print(f"len: {len(s)}")
print(f"peek: {s.peek()}")
print(f"s[0]: {s[0]}")
print(f"3 in s: {3 in s}")
print(f"9 in s: {9 in s}")

print("\nIterate:")
for item in s:
    print(f"  {item}", end="")
print()

print("\nReversed:")
for item in reversed(s):
    print(f"  {item}", end="")
print()

s[0] = 99
print(f"\nหลัง s[0]=99: {s}")

popped = s.pop()
print(f"Popped: {popped}")
print(f"Stack: {s}")
```

### 4.5 Context Manager Methods

```python
class DatabaseConnection:
    """Database connection ด้วย context manager"""
    
    def __init__(self, host: str, port: int, database: str):
        self.host = host
        self.port = port
        self.database = database
        self._connected = False
        self._queries = []
    
    def __enter__(self) -> "DatabaseConnection":
        """เปิด connection"""
        print(f"เชื่อมต่อ {self.host}:{self.port}/{self.database}")
        self._connected = True
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb) -> bool:
        """ปิด connection"""
        if exc_type:
            print(f"Rollback เนื่องจาก {exc_type.__name__}: {exc_val}")
        else:
            print(f"Commit {len(self._queries)} queries")
        
        self._connected = False
        print("ปิด connection แล้ว")
        return False  # propagate exceptions
    
    def execute(self, query: str) -> dict:
        """ส่ง query"""
        if not self._connected:
            raise RuntimeError("ยังไม่ได้เชื่อมต่อ")
        
        self._queries.append(query)
        # จำลอง query execution
        return {"query": query, "rows_affected": 1}
    
    def __repr__(self) -> str:
        status = "connected" if self._connected else "disconnected"
        return f"DB({self.database}@{self.host}, {status})"

# ทดสอบ
with DatabaseConnection("localhost", 5432, "myapp") as db:
    result = db.execute("INSERT INTO users VALUES (...)")
    result = db.execute("UPDATE products SET price = 100")
    print(f"Queries: {len(db._queries)}")

print()

# ทดสอบกับ exception
try:
    with DatabaseConnection("localhost", 5432, "myapp") as db:
        db.execute("SELECT * FROM users")
        raise ValueError("ข้อมูลผิดพลาด")
        db.execute("DELETE FROM temp")  # ไม่ถูกรัน
except ValueError:
    print("จัดการ exception นอก with block")
```

---

## 5. Class Inheritance (เบื้องต้น)

```python
# Inheritance - class สืบทอดจาก class อื่น
class Animal:
    """Base class สำหรับสัตว์ทุกชนิด"""
    
    def __init__(self, name: str, sound: str):
        self.name = name
        self.sound = sound
        self.alive = True
    
    def speak(self) -> str:
        return f"{self.name}: {self.sound}!"
    
    def eat(self, food: str) -> str:
        return f"{self.name} กิน {food}"
    
    def __str__(self) -> str:
        return f"{type(self).__name__}({self.name!r})"
    
    def __repr__(self) -> str:
        return f"{type(self).__name__}(name={self.name!r})"

class Dog(Animal):
    """Dog สืบทอดจาก Animal"""
    
    def __init__(self, name: str, breed: str):
        super().__init__(name, "โฮ่ง")  # เรียก parent's __init__
        self.breed = breed
        self._tricks = []
    
    # Override method
    def speak(self) -> str:
        return f"{self.name} ({self.breed}): โฮ่ง โฮ่ง!"
    
    # เพิ่ม method ใหม่
    def learn_trick(self, trick: str) -> None:
        self._tricks.append(trick)
    
    def perform(self) -> str:
        if self._tricks:
            return f"{self.name} แสดง: {', '.join(self._tricks)}"
        return f"{self.name} ไม่รู้ trick"

class Cat(Animal):
    """Cat สืบทอดจาก Animal"""
    
    def __init__(self, name: str, indoor: bool = True):
        super().__init__(name, "เมี๊ยว")
        self.indoor = indoor
    
    def speak(self) -> str:
        return f"{self.name}: เมี๊ยว~"
    
    def purr(self) -> str:
        return f"{self.name}: ครื้อๆๆๆ"

# ทดสอบ
dog = Dog("Max", "Golden Retriever")
cat = Cat("Luna")

print(dog.speak())
print(cat.speak())
print(cat.purr())

dog.learn_trick("นั่ง")
dog.learn_trick("ล้มตาย")
print(dog.perform())

print(f"\ndog กิน: {dog.eat('กระดูก')}")
print(f"cat กิน: {cat.eat('ปลา')}")

print(f"\n{dog}")
print(f"{cat}")

# isinstance และ issubclass
print(f"\ndog is Animal: {isinstance(dog, Animal)}")  # True
print(f"dog is Dog: {isinstance(dog, Dog)}")          # True
print(f"dog is Cat: {isinstance(dog, Cat)}")          # False
print(f"Dog is subclass of Animal: {issubclass(Dog, Animal)}")  # True

# Polymorphism
animals = [Dog("Rex", "Labrador"), Cat("Whiskers"), Dog("Buddy", "Poodle")]
print("\nPolymorphism:")
for animal in animals:
    print(f"  {animal.speak()}")
```

---

## 6. ตัวอย่างโปรแกรมจริง: Bank Account System

```python
"""
ระบบบัญชีธนาคาร OOP
"""
from datetime import datetime
from typing import List, Optional
import uuid

class Transaction:
    """บันทึก transaction"""
    
    def __init__(self, type_: str, amount: float, description: str = ""):
        self.id = str(uuid.uuid4())[:8]
        self.type = type_
        self.amount = amount
        self.description = description
        self.timestamp = datetime.now()
    
    def __str__(self) -> str:
        sign = "+" if self.type == "credit" else "-"
        return (f"[{self.timestamp.strftime('%Y-%m-%d %H:%M')}] "
                f"{sign}{self.amount:,.2f} บาท "
                f"({self.description})")

class BankAccount:
    """บัญชีธนาคาร"""
    
    MINIMUM_BALANCE = 100.0
    OVERDRAFT_FEE = 50.0
    
    def __init__(self, owner: str, account_number: str = None, 
                 initial_balance: float = 0.0):
        self.owner = owner
        self.account_number = account_number or f"ACC{uuid.uuid4().hex[:8].upper()}"
        self._balance = 0.0
        self._transactions: List[Transaction] = []
        self.active = True
        
        if initial_balance > 0:
            self._credit(initial_balance, "เปิดบัญชี")
    
    @property
    def balance(self) -> float:
        return self._balance
    
    def _credit(self, amount: float, description: str) -> Transaction:
        """เพิ่มเงินเข้าบัญชี"""
        t = Transaction("credit", amount, description)
        self._balance += amount
        self._transactions.append(t)
        return t
    
    def _debit(self, amount: float, description: str) -> Transaction:
        """ถอนเงินออกจากบัญชี"""
        t = Transaction("debit", amount, description)
        self._balance -= amount
        self._transactions.append(t)
        return t
    
    def deposit(self, amount: float, description: str = "ฝากเงิน") -> Transaction:
        """ฝากเงิน"""
        if not self.active:
            raise RuntimeError("บัญชีถูกปิดแล้ว")
        if amount <= 0:
            raise ValueError("จำนวนเงินต้องมากกว่า 0")
        return self._credit(amount, description)
    
    def withdraw(self, amount: float, description: str = "ถอนเงิน") -> Transaction:
        """ถอนเงิน"""
        if not self.active:
            raise RuntimeError("บัญชีถูกปิดแล้ว")
        if amount <= 0:
            raise ValueError("จำนวนเงินต้องมากกว่า 0")
        
        if self._balance - amount < self.MINIMUM_BALANCE:
            # ถอนเกิน - คิดค่าธรรมเนียม
            if self._balance - amount < 0:
                raise ValueError(f"ยอดเงินไม่พอ (มี {self._balance:.2f} บาท)")
            self._debit(self.OVERDRAFT_FEE, "ค่าธรรมเนียม (ยอดต่ำกว่าขั้นต่ำ)")
        
        return self._debit(amount, description)
    
    def transfer(self, target: "BankAccount", amount: float, description: str = "โอนเงิน") -> bool:
        """โอนเงินไปยังบัญชีอื่น"""
        self.withdraw(amount, f"โอนไป {target.account_number}: {description}")
        target.deposit(amount, f"รับจาก {self.account_number}: {description}")
        return True
    
    def get_statement(self, last_n: int = None) -> str:
        """แสดง statement"""
        transactions = self._transactions
        if last_n:
            transactions = transactions[-last_n:]
        
        lines = [
            f"{'=' * 55}",
            f"Statement: {self.owner}",
            f"Account: {self.account_number}",
            f"{'=' * 55}",
        ]
        
        for t in transactions:
            lines.append(f"  {t}")
        
        lines.extend([
            f"{'=' * 55}",
            f"  ยอดคงเหลือ: {self._balance:,.2f} บาท",
            f"{'=' * 55}",
        ])
        
        return "\n".join(lines)
    
    def __str__(self) -> str:
        return f"Account({self.owner}, {self.account_number}, {self._balance:,.2f}฿)"
    
    def __repr__(self) -> str:
        return f"BankAccount(owner={self.owner!r}, number={self.account_number!r})"
    
    def __eq__(self, other) -> bool:
        if isinstance(other, BankAccount):
            return self.account_number == other.account_number
        return NotImplemented
    
    def __lt__(self, other) -> bool:
        if isinstance(other, BankAccount):
            return self._balance < other._balance
        return NotImplemented
    
    def __add__(self, other):
        """รวมยอดของ 2 บัญชี - คืน float"""
        if isinstance(other, BankAccount):
            return self._balance + other._balance
        return NotImplemented
    
    def __float__(self) -> float:
        return self._balance

# ทดสอบ
alice_acc = BankAccount("Alice", initial_balance=10000)
bob_acc = BankAccount("Bob", initial_balance=5000)

print(alice_acc)
print(bob_acc)

# Transactions
alice_acc.deposit(2000, "เงินเดือน")
alice_acc.withdraw(500, "ค่าอาหาร")
alice_acc.withdraw(1200, "ค่าเช่า")

bob_acc.deposit(3000, "โบนัส")
bob_acc.withdraw(200, "ค่าโทรศัพท์")

# โอนเงิน
alice_acc.transfer(bob_acc, 1000, "คืนเงินที่ยืม")

# Statement
print("\n" + alice_acc.get_statement(last_n=5))
print("\n" + bob_acc.get_statement())

# Magic methods
print(f"\nยอดรวม: {alice_acc + bob_acc:,.2f} บาท")
print(f"float(alice): {float(alice_acc):,.2f}")

accounts = [alice_acc, bob_acc, BankAccount("Charlie", initial_balance=8000)]
richest = max(accounts)
print(f"บัญชีที่มียอดสูงสุด: {richest}")
```

---

## 7. Exercises

### Exercise 1: Product Catalog

```python
"""
สร้างระบบ Product Catalog ด้วย OOP:
1. Product class พร้อม properties
2. Inventory class สำหรับจัดการสต็อก
3. Magic methods: __str__, __repr__, __eq__, __lt__, __len__
"""
from typing import List, Optional
from datetime import datetime

class Product:
    def __init__(self, id: str, name: str, price: float, 
                 category: str, stock: int = 0):
        self.id = id
        self.name = name
        self._price = price
        self.category = category
        self._stock = stock
        self.created_at = datetime.now()
    
    @property
    def price(self) -> float:
        return self._price
    
    @price.setter
    def price(self, value: float) -> None:
        if value < 0:
            raise ValueError("ราคาต้องไม่ติดลบ")
        self._price = value
    
    @property
    def stock(self) -> int:
        return self._stock
    
    @property
    def in_stock(self) -> bool:
        return self._stock > 0
    
    def restock(self, qty: int) -> None:
        if qty <= 0:
            raise ValueError("จำนวนต้องเป็นบวก")
        self._stock += qty
    
    def sell(self, qty: int = 1) -> None:
        if qty > self._stock:
            raise ValueError(f"สต็อกไม่พอ (มี {self._stock})")
        self._stock -= qty
    
    def __str__(self) -> str:
        stock_status = f"{self._stock} ชิ้น" if self.in_stock else "หมด"
        return f"[{self.id}] {self.name} - {self._price:,.2f}฿ ({stock_status})"
    
    def __repr__(self) -> str:
        return f"Product(id={self.id!r}, name={self.name!r}, price={self._price})"
    
    def __eq__(self, other) -> bool:
        return isinstance(other, Product) and self.id == other.id
    
    def __lt__(self, other) -> bool:
        return isinstance(other, Product) and self._price < other._price
    
    def __hash__(self) -> int:
        return hash(self.id)

class Inventory:
    def __init__(self):
        self._products = {}
    
    def add_product(self, product: Product) -> None:
        self._products[product.id] = product
    
    def get_product(self, product_id: str) -> Optional[Product]:
        return self._products.get(product_id)
    
    def search(self, query: str = "", category: str = "") -> List[Product]:
        results = list(self._products.values())
        if query:
            results = [p for p in results if query.lower() in p.name.lower()]
        if category:
            results = [p for p in results if p.category == category]
        return sorted(results)
    
    def low_stock(self, threshold: int = 5) -> List[Product]:
        return [p for p in self._products.values() if p.stock <= threshold]
    
    def __len__(self) -> int:
        return len(self._products)
    
    def __contains__(self, product_id: str) -> bool:
        return product_id in self._products
    
    def __iter__(self):
        return iter(self._products.values())

# ทดสอบ
inv = Inventory()

products = [
    Product("P001", "Laptop Pro", 45000, "Electronics", 10),
    Product("P002", "Mouse Wireless", 599, "Electronics", 3),
    Product("P003", "Mechanical Keyboard", 2499, "Electronics", 8),
    Product("P004", "Office Chair", 8500, "Furniture", 5),
    Product("P005", "Desk Lamp", 1299, "Furniture", 12),
]

for p in products:
    inv.add_product(p)

print(f"สินค้าทั้งหมด: {len(inv)} รายการ")
print()

# แสดงทุกชิ้น
print("=== สินค้าทั้งหมด (เรียงตามราคา) ===")
for p in inv.search():
    print(f"  {p}")

# ค้นหา
print("\n=== ค้นหา 'laptop' ===")
for p in inv.search(query="laptop"):
    print(f"  {p}")

# Low stock
print("\n=== สต็อกต่ำ (<=5) ===")
for p in inv.low_stock(5):
    print(f"  {p}")

# ซื้อสินค้า
laptop = inv.get_product("P001")
laptop.sell(3)
print(f"\nหลังขาย 3 เครื่อง: {laptop}")
```

### Exercise 2: Matrix Class

```python
"""
สร้าง Matrix class พร้อม magic methods:
- __add__, __sub__, __mul__ (matrix multiply)
- __getitem__, __setitem__
- __len__, __iter__
- __str__, __repr__
- __eq__
"""
from typing import List, Union

class Matrix:
    def __init__(self, data: List[List[float]]):
        if not data or not data[0]:
            raise ValueError("Matrix ต้องมีข้อมูล")
        
        rows = len(data)
        cols = len(data[0])
        
        if any(len(row) != cols for row in data):
            raise ValueError("แต่ละแถวต้องมีจำนวน column เท่ากัน")
        
        self._data = [[float(x) for x in row] for row in data]
        self.rows = rows
        self.cols = cols
    
    @classmethod
    def zeros(cls, rows: int, cols: int) -> "Matrix":
        return cls([[0] * cols for _ in range(rows)])
    
    @classmethod
    def identity(cls, n: int) -> "Matrix":
        data = [[1 if i == j else 0 for j in range(n)] for i in range(n)]
        return cls(data)
    
    def __getitem__(self, key):
        if isinstance(key, tuple):
            row, col = key
            return self._data[row][col]
        return self._data[key]
    
    def __setitem__(self, key, value):
        if isinstance(key, tuple):
            row, col = key
            self._data[row][col] = float(value)
        else:
            self._data[key] = [float(x) for x in value]
    
    def __len__(self) -> int:
        return self.rows
    
    def __iter__(self):
        return iter(self._data)
    
    def __add__(self, other: "Matrix") -> "Matrix":
        if self.rows != other.rows or self.cols != other.cols:
            raise ValueError("Matrix ขนาดต้องเท่ากันเพื่อบวก")
        return Matrix([[self[i][j] + other[i][j] 
                        for j in range(self.cols)] 
                       for i in range(self.rows)])
    
    def __sub__(self, other: "Matrix") -> "Matrix":
        if self.rows != other.rows or self.cols != other.cols:
            raise ValueError("Matrix ขนาดต้องเท่ากันเพื่อลบ")
        return Matrix([[self[i][j] - other[i][j] 
                        for j in range(self.cols)] 
                       for i in range(self.rows)])
    
    def __mul__(self, other: Union["Matrix", float]) -> "Matrix":
        if isinstance(other, (int, float)):
            return Matrix([[self[i][j] * other 
                            for j in range(self.cols)] 
                           for i in range(self.rows)])
        if isinstance(other, Matrix):
            if self.cols != other.rows:
                raise ValueError(f"ไม่สามารถคูณ {self.rows}x{self.cols} กับ {other.rows}x{other.cols}")
            return Matrix([
                [sum(self[i][k] * other[k][j] for k in range(self.cols))
                 for j in range(other.cols)]
                for i in range(self.rows)
            ])
        return NotImplemented
    
    def __rmul__(self, scalar: float) -> "Matrix":
        return self.__mul__(scalar)
    
    def __eq__(self, other) -> bool:
        if not isinstance(other, Matrix):
            return NotImplemented
        return self._data == other._data
    
    def transpose(self) -> "Matrix":
        return Matrix([[self[i][j] for i in range(self.rows)] 
                       for j in range(self.cols)])
    
    def __str__(self) -> str:
        lines = []
        for row in self._data:
            row_str = "  ".join(f"{x:6.2f}" for x in row)
            lines.append(f"[ {row_str} ]")
        return "\n".join(lines)
    
    def __repr__(self) -> str:
        return f"Matrix({self._data})"

# ทดสอบ
A = Matrix([[1, 2], [3, 4]])
B = Matrix([[5, 6], [7, 8]])
I = Matrix.identity(2)

print("A:")
print(A)
print("\nB:")
print(B)
print("\nA + B:")
print(A + B)
print("\nA * B:")
print(A * B)
print("\nA * 2:")
print(A * 2)
print("\nA^T (transpose):")
print(A.transpose())
print(f"\nI = Identity:\n{I}")
print(f"\nA == A: {A == A}")
print(f"A == B: {A == B}")
```

---

## 8. สรุป Part 015

### สิ่งที่เรียนรู้:

✅ **class** - สร้าง class  
✅ **\_\_init\_\_** - constructor  
✅ **self** - reference ถึง instance ปัจจุบัน  
✅ **Instance variables** - แต่ละ object มีเป็นของตัวเอง  
✅ **Class variables** - ใช้ร่วมกันทุก instance  
✅ **Instance methods** - รับ self  
✅ **@classmethod** - รับ cls  
✅ **@staticmethod** - ไม่รับ self/cls  
✅ **@property** - getter/setter/deleter  
✅ **Magic methods** - \_\_str\_\_, \_\_repr\_\_, \_\_add\_\_, \_\_len\_\_, \_\_eq\_\_  
✅ **Inheritance** - class Child(Parent)  
✅ **super()** - เรียก parent method  

### Quick Reference:

```python
class MyClass:
    class_var = "shared"      # class variable
    
    def __init__(self, x):
        self.x = x            # instance variable
    
    def instance_method(self):    # รับ self
        return self.x
    
    @classmethod
    def class_method(cls):        # รับ cls
        return cls.class_var
    
    @staticmethod
    def static_method():          # ไม่รับ self/cls
        return "static"
    
    @property
    def computed(self):           # getter
        return self.x * 2
    
    @computed.setter
    def computed(self, val):      # setter
        self.x = val / 2
    
    def __str__(self): return f"MyClass({self.x})"
    def __repr__(self): return f"MyClass(x={self.x!r})"
    def __len__(self): return self.x
    def __add__(self, other): return MyClass(self.x + other.x)
    def __eq__(self, other): return self.x == other.x
    def __lt__(self, other): return self.x < other.x
    def __contains__(self, item): return item == self.x
    def __enter__(self): return self
    def __exit__(self, *args): return False
```

---

## ➡️ ถัดไป: Part 016 - OOP ขั้นสูง (Inheritance, ABC, MRO)

ใน Part ถัดไป เราจะเรียนรู้:
- Multiple Inheritance และ MRO
- Abstract Base Classes (ABC)
- Mixins
- Dataclasses
- Protocols (Duck Typing)

---

*Part 015/100+ | Python Course - Beginner to World-Class*
