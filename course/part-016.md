# Part 016 - OOP Inheritance (การสืบทอดคลาส)

## เป้าหมาย
- เข้าใจหลักการ Single Inheritance และ Multiple Inheritance
- ใช้งาน `super()` เพื่อเรียกเมธอดของ parent class
- เข้าใจ Method Resolution Order (MRO)
- สร้าง Abstract Classes ด้วย ABC
- ใช้ Mixin Classes เพื่อเพิ่มความสามารถ

---

## 1. Single Inheritance (การสืบทอดชั้นเดียว)

การสืบทอดช่วยให้คลาสลูก (child class) รับคุณสมบัติและเมธอดจากคลาสแม่ (parent class)

```python
# คลาสพื้นฐาน (Base Class / Parent Class)
class Animal:
    def __init__(self, name: str, sound: str):
        self.name = name
        self.sound = sound
        self.is_alive = True
    
    def speak(self):
        """เมธอดสำหรับส่งเสียง"""
        return f"{self.name} says {self.sound}"
    
    def eat(self, food: str):
        """เมธอดสำหรับกิน"""
        return f"{self.name} is eating {food}"
    
    def __str__(self):
        return f"Animal({self.name})"
    
    def __repr__(self):
        return f"Animal(name={self.name!r}, sound={self.sound!r})"


# คลาสลูก (Child Class) สืบทอดจาก Animal
class Dog(Animal):
    def __init__(self, name: str, breed: str):
        # เรียก __init__ ของ parent class
        super().__init__(name, sound="Woof")
        self.breed = breed
        self.tricks = []
    
    def learn_trick(self, trick: str):
        """สอนเทคนิคให้หมา"""
        self.tricks.append(trick)
        return f"{self.name} learned {trick}!"
    
    def show_tricks(self):
        """แสดงเทคนิคทั้งหมด"""
        if not self.tricks:
            return f"{self.name} doesn't know any tricks yet"
        return f"{self.name} can: {', '.join(self.tricks)}"
    
    def fetch(self, item: str):
        """เมธอดเฉพาะของหมา"""
        return f"{self.name} fetches the {item}!"
    
    def __str__(self):
        return f"Dog({self.name}, {self.breed})"


class Cat(Animal):
    def __init__(self, name: str, indoor: bool = True):
        super().__init__(name, sound="Meow")
        self.indoor = indoor
        self.lives = 9
    
    def purr(self):
        """แมวร้องเป็นเสียงครื้น"""
        return f"{self.name} purrs contentedly..."
    
    def scratch(self, target: str):
        """แมวข่วน"""
        return f"{self.name} scratches {target}!"
    
    def __str__(self):
        location = "indoor" if self.indoor else "outdoor"
        return f"Cat({self.name}, {location})"


# ทดสอบ Single Inheritance
dog = Dog("Buddy", "Golden Retriever")
cat = Cat("Whiskers")

print(dog.speak())          # Buddy says Woof  (สืบทอดจาก Animal)
print(dog.eat("kibble"))    # Buddy is eating kibble  (สืบทอดจาก Animal)
print(dog.fetch("ball"))    # Buddy fetches the ball!  (เมธอดของ Dog)

dog.learn_trick("sit")
dog.learn_trick("shake")
print(dog.show_tricks())    # Buddy can: sit, shake

print(cat.speak())          # Whiskers says Meow
print(cat.purr())           # Whiskers purrs contentedly...

# ตรวจสอบ inheritance
print(isinstance(dog, Dog))     # True
print(isinstance(dog, Animal))  # True - dog ก็เป็น Animal ด้วย
print(isinstance(cat, Dog))     # False
print(issubclass(Dog, Animal))  # True
print(issubclass(Animal, Dog))  # False
```

---

## 2. Method Overriding (การ Override เมธอด)

คลาสลูกสามารถ override เมธอดของคลาสแม่ได้

```python
class Shape:
    """คลาสพื้นฐานสำหรับรูปทรงเรขาคณิต"""
    
    def __init__(self, color: str = "white"):
        self.color = color
    
    def area(self) -> float:
        """คำนวณพื้นที่ - ต้อง override ในคลาสลูก"""
        raise NotImplementedError("Subclasses must implement area()")
    
    def perimeter(self) -> float:
        """คำนวณเส้นรอบวง - ต้อง override ในคลาสลูก"""
        raise NotImplementedError("Subclasses must implement perimeter()")
    
    def describe(self) -> str:
        """อธิบายรูปทรง"""
        return (f"{self.__class__.__name__} - "
                f"Color: {self.color}, "
                f"Area: {self.area():.2f}, "
                f"Perimeter: {self.perimeter():.2f}")
    
    def __str__(self):
        return f"{self.__class__.__name__}(color={self.color})"


import math

class Circle(Shape):
    def __init__(self, radius: float, color: str = "white"):
        super().__init__(color)
        self.radius = radius
    
    def area(self) -> float:
        """Override: คำนวณพื้นที่วงกลม"""
        return math.pi * self.radius ** 2
    
    def perimeter(self) -> float:
        """Override: คำนวณเส้นรอบวง"""
        return 2 * math.pi * self.radius
    
    def diameter(self) -> float:
        """เมธอดเฉพาะของวงกลม"""
        return self.radius * 2


class Rectangle(Shape):
    def __init__(self, width: float, height: float, color: str = "white"):
        super().__init__(color)
        self.width = width
        self.height = height
    
    def area(self) -> float:
        """Override: คำนวณพื้นที่สี่เหลี่ยม"""
        return self.width * self.height
    
    def perimeter(self) -> float:
        """Override: คำนวณเส้นรอบวง"""
        return 2 * (self.width + self.height)
    
    def diagonal(self) -> float:
        """คำนวณเส้นทแยงมุม"""
        return math.sqrt(self.width ** 2 + self.height ** 2)


class Triangle(Shape):
    def __init__(self, a: float, b: float, c: float, color: str = "white"):
        super().__init__(color)
        self.a = a  # ด้าน a
        self.b = b  # ด้าน b
        self.c = c  # ด้าน c
    
    def area(self) -> float:
        """Heron's formula สำหรับพื้นที่สามเหลี่ยม"""
        s = (self.a + self.b + self.c) / 2  # กึ่งเส้นรอบวง
        return math.sqrt(s * (s - self.a) * (s - self.b) * (s - self.c))
    
    def perimeter(self) -> float:
        """Override: คำนวณเส้นรอบวง"""
        return self.a + self.b + self.c


# ทดสอบ Method Overriding
shapes = [
    Circle(5, "red"),
    Rectangle(4, 6, "blue"),
    Triangle(3, 4, 5, "green")
]

for shape in shapes:
    print(shape.describe())

# Output:
# Circle - Color: red, Area: 78.54, Perimeter: 31.42
# Rectangle - Color: blue, Area: 24.00, Perimeter: 20.00
# Triangle - Color: green, Area: 6.00, Perimeter: 12.00
```

---

## 3. super() - การเรียกเมธอดของ Parent Class

```python
class Employee:
    """คลาสพนักงาน"""
    
    def __init__(self, name: str, salary: float, department: str):
        self.name = name
        self.salary = salary
        self.department = department
    
    def get_info(self) -> dict:
        """ข้อมูลพนักงาน"""
        return {
            "name": self.name,
            "salary": self.salary,
            "department": self.department
        }
    
    def give_raise(self, amount: float):
        """ขึ้นเงินเดือน"""
        self.salary += amount
        return f"{self.name} got a raise of {amount:,.0f}. New salary: {self.salary:,.0f}"
    
    def __str__(self):
        return f"Employee({self.name}, {self.department})"


class Manager(Employee):
    """คลาสผู้จัดการ สืบทอดจาก Employee"""
    
    def __init__(self, name: str, salary: float, department: str, 
                 team_size: int, budget: float):
        # เรียก __init__ ของ Employee
        super().__init__(name, salary, department)
        self.team_size = team_size
        self.budget = budget
        self.direct_reports = []
    
    def get_info(self) -> dict:
        """Override: เพิ่มข้อมูลของผู้จัดการ"""
        # เรียก get_info ของ parent แล้วเพิ่มข้อมูล
        info = super().get_info()
        info.update({
            "role": "Manager",
            "team_size": self.team_size,
            "budget": self.budget,
            "direct_reports": len(self.direct_reports)
        })
        return info
    
    def add_report(self, employee: Employee):
        """เพิ่มพนักงานในทีม"""
        self.direct_reports.append(employee)
        return f"{employee.name} now reports to {self.name}"
    
    def give_raise(self, amount: float):
        """Override: ผู้จัดการได้ขึ้นเงินเดือนเพิ่มพิเศษ 20%"""
        # เรียก give_raise ของ parent
        result = super().give_raise(amount * 1.2)  # โบนัส 20%
        return f"[Manager bonus applied] {result}"
    
    def approve_expense(self, amount: float) -> str:
        """อนุมัติค่าใช้จ่าย"""
        if amount <= self.budget:
            self.budget -= amount
            return f"Expense of {amount:,.0f} approved. Remaining budget: {self.budget:,.0f}"
        return f"Expense of {amount:,.0f} exceeds budget of {self.budget:,.0f}!"


class Director(Manager):
    """คลาสผู้อำนวยการ สืบทอดจาก Manager"""
    
    def __init__(self, name: str, salary: float, department: str,
                 team_size: int, budget: float, division: str):
        super().__init__(name, salary, department, team_size, budget)
        self.division = division
    
    def get_info(self) -> dict:
        """Override: เพิ่มข้อมูลของผู้อำนวยการ"""
        info = super().get_info()  # ได้ข้อมูลจาก Manager
        info.update({
            "role": "Director",
            "division": self.division
        })
        return info


# ทดสอบ super()
emp = Employee("Alice", 50000, "Engineering")
mgr = Manager("Bob", 80000, "Engineering", 10, 500000)
dir = Director("Carol", 150000, "Engineering", 50, 5000000, "Technology")

mgr.add_report(emp)

print(emp.get_info())
print()
print(mgr.get_info())
print()
print(dir.get_info())
print()

# ทดสอบ give_raise
print(emp.give_raise(5000))
print(mgr.give_raise(10000))  # ได้โบนัส 20% เพิ่ม
```

---

## 4. Multiple Inheritance (การสืบทอดหลายคลาส)

```python
# Python รองรับการสืบทอดจากหลายคลาสพร้อมกัน

class Flyable:
    """Mixin สำหรับสิ่งที่บินได้"""
    
    def fly(self) -> str:
        return f"{self.__class__.__name__} is flying!"
    
    def land(self) -> str:
        return f"{self.__class__.__name__} is landing!"
    
    def altitude(self) -> str:
        return f"{self.__class__.__name__} is at high altitude"


class Swimmable:
    """Mixin สำหรับสิ่งที่ว่ายน้ำได้"""
    
    def swim(self) -> str:
        return f"{self.__class__.__name__} is swimming!"
    
    def dive(self) -> str:
        return f"{self.__class__.__name__} is diving!"
    
    def surface(self) -> str:
        return f"{self.__class__.__name__} is surfacing!"


class Walkable:
    """Mixin สำหรับสิ่งที่เดินได้"""
    
    def walk(self) -> str:
        return f"{self.__class__.__name__} is walking!"
    
    def run(self) -> str:
        return f"{self.__class__.__name__} is running!"


# Duck สืบทอดจากหลายคลาส
class Duck(Flyable, Swimmable, Walkable):
    """เป็ดที่สามารถบิน ว่ายน้ำ และเดินได้"""
    
    def __init__(self, name: str):
        self.name = name
    
    def quack(self) -> str:
        return f"{self.name}: Quack quack!"
    
    def __str__(self):
        return f"Duck({self.name})"


# Penguin บินไม่ได้ แต่ว่ายน้ำได้
class Penguin(Swimmable, Walkable):
    """นกเพนกวินว่ายน้ำและเดินได้ แต่บินไม่ได้"""
    
    def __init__(self, name: str):
        self.name = name
    
    def slide(self) -> str:
        return f"{self.name} slides on ice!"


duck = Duck("Donald")
print(duck.fly())     # Donald is flying!
print(duck.swim())    # Donald is swimming!
print(duck.walk())    # Donald is walking!
print(duck.quack())   # Donald: Quack quack!

penguin = Penguin("Pingu")
print(penguin.swim())   # Penguin is swimming!
print(penguin.walk())   # Penguin is walking!
print(penguin.slide())  # Pingu slides on ice!

# ตรวจสอบว่าเป็ดมีความสามารถอะไรบ้าง
print(isinstance(duck, Flyable))    # True
print(isinstance(duck, Swimmable))  # True
print(isinstance(duck, Walkable))   # True
print(isinstance(penguin, Flyable)) # False
```

---

## 5. Method Resolution Order (MRO)

```python
# MRO คือลำดับที่ Python ค้นหาเมธอดในกรณีที่มีหลายคลาส

class A:
    def method(self):
        return "A.method"
    
    def greet(self):
        return "Hello from A"


class B(A):
    def method(self):
        return f"B.method -> {super().method()}"


class C(A):
    def method(self):
        return f"C.method -> {super().method()}"


class D(B, C):
    def method(self):
        return f"D.method -> {super().method()}"


# ดู MRO
print(D.__mro__)
# (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)

# หรือใช้ mro() method
for cls in D.mro():
    print(cls.__name__, end=" -> ")
# D -> B -> C -> A -> object ->

d = D()
print(d.method())
# D.method -> B.method -> C.method -> A.method

# ตัวอย่างที่ซับซ้อนขึ้น: Diamond Problem
class Vehicle:
    def start(self):
        return "Vehicle starting..."
    
    def fuel_type(self):
        return "gasoline"


class Car(Vehicle):
    def start(self):
        result = super().start()
        return f"Car: {result}"
    
    def fuel_type(self):
        return "gasoline/electric"


class Truck(Vehicle):
    def start(self):
        result = super().start()
        return f"Truck: {result}"
    
    def fuel_type(self):
        return "diesel"


class PickupTruck(Car, Truck):
    def start(self):
        result = super().start()
        return f"PickupTruck: {result}"


pickup = PickupTruck()
print(PickupTruck.__mro__)
print(pickup.start())
# PickupTruck: Car: Truck: Vehicle starting...

# Python ใช้ C3 linearization สำหรับ MRO
print(PickupTruck.mro())
```

---

## 6. Abstract Classes (ABC)

```python
from abc import ABC, abstractmethod
from typing import List

class DatabaseConnector(ABC):
    """Abstract class สำหรับ Database Connection"""
    
    def __init__(self, host: str, port: int, database: str):
        self.host = host
        self.port = port
        self.database = database
        self._connection = None
    
    @abstractmethod
    def connect(self) -> bool:
        """เชื่อมต่อกับ database - ต้อง implement"""
        pass
    
    @abstractmethod
    def disconnect(self):
        """ยกเลิกการเชื่อมต่อ - ต้อง implement"""
        pass
    
    @abstractmethod
    def execute(self, query: str) -> List:
        """รัน query - ต้อง implement"""
        pass
    
    # เมธอดที่มี implementation แล้ว (ไม่ต้อง override)
    def get_connection_string(self) -> str:
        """สร้าง connection string"""
        return f"{self.host}:{self.port}/{self.database}"
    
    def is_connected(self) -> bool:
        """ตรวจสอบการเชื่อมต่อ"""
        return self._connection is not None
    
    @abstractmethod
    def __enter__(self):
        """Context manager: enter"""
        pass
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        """Context manager: exit"""
        self.disconnect()
        return False


class PostgreSQLConnector(DatabaseConnector):
    """Implementation สำหรับ PostgreSQL"""
    
    def connect(self) -> bool:
        """เชื่อมต่อ PostgreSQL"""
        # ในตัวอย่างนี้ simulate การเชื่อมต่อ
        print(f"Connecting to PostgreSQL at {self.get_connection_string()}")
        self._connection = {"type": "postgresql", "status": "connected"}
        return True
    
    def disconnect(self):
        """ยกเลิกการเชื่อมต่อ"""
        if self._connection:
            print("Disconnecting from PostgreSQL")
            self._connection = None
    
    def execute(self, query: str) -> List:
        """รัน query บน PostgreSQL"""
        if not self.is_connected():
            raise RuntimeError("Not connected to database!")
        print(f"PostgreSQL executing: {query}")
        return [{"result": "mock_data"}]
    
    def __enter__(self):
        self.connect()
        return self


class MySQLConnector(DatabaseConnector):
    """Implementation สำหรับ MySQL"""
    
    def connect(self) -> bool:
        print(f"Connecting to MySQL at {self.get_connection_string()}")
        self._connection = {"type": "mysql", "status": "connected"}
        return True
    
    def disconnect(self):
        if self._connection:
            print("Disconnecting from MySQL")
            self._connection = None
    
    def execute(self, query: str) -> List:
        if not self.is_connected():
            raise RuntimeError("Not connected to database!")
        print(f"MySQL executing: {query}")
        return [{"result": "mock_data"}]
    
    def __enter__(self):
        self.connect()
        return self


# ทดสอบ ABC
# ไม่สามารถสร้าง instance ของ ABC ได้โดยตรง
try:
    db = DatabaseConnector("localhost", 5432, "mydb")  # TypeError!
except TypeError as e:
    print(f"Error: {e}")

# ใช้งานผ่าน concrete classes
with PostgreSQLConnector("localhost", 5432, "myapp") as pg:
    results = pg.execute("SELECT * FROM users")
    print(results)

with MySQLConnector("localhost", 3306, "myapp") as mysql:
    results = mysql.execute("SELECT * FROM products")
    print(results)

# ตรวจสอบ abstract methods
print(DatabaseConnector.__abstractmethods__)  
# frozenset({'connect', 'disconnect', 'execute', '__enter__'})
```

---

## 7. Mixin Classes (การใช้งาน Mixin)

```python
import json
from datetime import datetime
from typing import Any, Dict

# Mixin คือคลาสที่ให้ functionality เพิ่มเติม
# ไม่ควรใช้เป็น standalone class

class TimestampMixin:
    """Mixin สำหรับเพิ่ม timestamp ให้กับ object"""
    
    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
    
    def touch(self):
        """อัปเดต updated_at"""
        self.updated_at = datetime.now()
    
    @property
    def age(self) -> float:
        """คำนวณอายุของ object เป็นวินาที"""
        if hasattr(self, 'created_at'):
            return (datetime.now() - self.created_at).total_seconds()
        return 0


class SerializableMixin:
    """Mixin สำหรับ serialize/deserialize object"""
    
    def to_dict(self) -> Dict[str, Any]:
        """แปลง object เป็น dictionary"""
        result = {}
        for key, value in self.__dict__.items():
            if isinstance(value, datetime):
                result[key] = value.isoformat()
            else:
                result[key] = value
        return result
    
    def to_json(self) -> str:
        """แปลง object เป็น JSON string"""
        return json.dumps(self.to_dict(), default=str, indent=2)
    
    @classmethod
    def from_dict(cls, data: Dict[str, Any]):
        """สร้าง object จาก dictionary"""
        obj = cls.__new__(cls)
        for key, value in data.items():
            setattr(obj, key, value)
        return obj


class ValidatorMixin:
    """Mixin สำหรับ validate ข้อมูล"""
    
    # subclass ควร define validators
    _validators = {}
    
    def validate(self) -> bool:
        """ตรวจสอบความถูกต้องของข้อมูล"""
        errors = []
        for field, validator in self.__class__._validators.items():
            value = getattr(self, field, None)
            if not validator(value):
                errors.append(f"Invalid value for {field}: {value}")
        
        if errors:
            raise ValueError("\n".join(errors))
        return True


class LoggableMixin:
    """Mixin สำหรับ logging"""
    
    _log = []
    
    def log(self, message: str):
        """บันทึก log"""
        entry = {
            "timestamp": datetime.now().isoformat(),
            "class": self.__class__.__name__,
            "message": message
        }
        self.__class__._log.append(entry)
        print(f"[{entry['timestamp']}] {entry['class']}: {message}")
    
    @classmethod
    def get_logs(cls):
        """ดู log ทั้งหมด"""
        return cls._log.copy()


# ใช้ Mixin กับ Business Class
class User(TimestampMixin, SerializableMixin, LoggableMixin):
    """คลาส User ที่ใช้ Mixin หลายตัว"""
    
    def __init__(self, user_id: int, username: str, email: str):
        self.user_id = user_id
        self.username = username
        self.email = email
        self.created_at = datetime.now()
        self.updated_at = datetime.now()
        self.log(f"User {username} created")
    
    def update_email(self, new_email: str):
        """อัปเดต email"""
        old_email = self.email
        self.email = new_email
        self.touch()  # อัปเดต updated_at จาก TimestampMixin
        self.log(f"Email updated from {old_email} to {new_email}")


class Product(TimestampMixin, SerializableMixin):
    """คลาส Product ที่ใช้ Mixin"""
    
    def __init__(self, product_id: int, name: str, price: float):
        self.product_id = product_id
        self.name = name
        self.price = price
        self.created_at = datetime.now()
        self.updated_at = datetime.now()
    
    def update_price(self, new_price: float):
        """อัปเดตราคา"""
        self.price = new_price
        self.touch()


# ทดสอบ Mixin
import time

user = User(1, "john_doe", "john@example.com")
time.sleep(0.1)
user.update_email("john.doe@example.com")

print(user.to_json())
print(f"\nUser age: {user.age:.2f} seconds")

product = Product(101, "Python Book", 500.0)
product.update_price(450.0)
print(product.to_dict())
```

---

## 8. ตัวอย่างจริง: ระบบจัดการพนักงาน

```python
from abc import ABC, abstractmethod
from datetime import datetime, date
from typing import List, Optional
import json

class PersonMixin:
    """Mixin สำหรับข้อมูลส่วนตัว"""
    
    def get_age(self) -> int:
        """คำนวณอายุ"""
        today = date.today()
        return today.year - self.birth_date.year - (
            (today.month, today.day) < (self.birth_date.month, self.birth_date.day)
        )
    
    def full_name(self) -> str:
        """ชื่อเต็ม"""
        return f"{self.first_name} {self.last_name}"


class Payable(ABC):
    """Abstract class สำหรับการจ่ายเงิน"""
    
    @abstractmethod
    def calculate_pay(self) -> float:
        """คำนวณเงินเดือน - ต้อง implement"""
        pass
    
    @abstractmethod
    def get_tax(self) -> float:
        """คำนวณภาษี - ต้อง implement"""
        pass
    
    def net_pay(self) -> float:
        """เงินเดือนหลังหักภาษี"""
        return self.calculate_pay() - self.get_tax()


class StaffMember(PersonMixin, Payable, ABC):
    """Base class สำหรับพนักงานทุกประเภท"""
    
    def __init__(self, employee_id: str, first_name: str, last_name: str,
                 birth_date: date, department: str):
        self.employee_id = employee_id
        self.first_name = first_name
        self.last_name = last_name
        self.birth_date = birth_date
        self.department = department
        self.hire_date = date.today()
        self.is_active = True
    
    def years_employed(self) -> int:
        """จำนวนปีที่ทำงาน"""
        today = date.today()
        return today.year - self.hire_date.year
    
    def get_info(self) -> dict:
        """ข้อมูลพนักงาน"""
        return {
            "employee_id": self.employee_id,
            "name": self.full_name(),
            "age": self.get_age(),
            "department": self.department,
            "years_employed": self.years_employed(),
            "gross_pay": self.calculate_pay(),
            "tax": self.get_tax(),
            "net_pay": self.net_pay(),
            "type": self.__class__.__name__
        }
    
    def __str__(self):
        return f"{self.__class__.__name__}({self.employee_id}: {self.full_name()})"


class FullTimeEmployee(StaffMember):
    """พนักงานประจำ"""
    TAX_RATE = 0.15  # ภาษี 15%
    
    def __init__(self, employee_id: str, first_name: str, last_name: str,
                 birth_date: date, department: str, monthly_salary: float):
        super().__init__(employee_id, first_name, last_name, birth_date, department)
        self.monthly_salary = monthly_salary
    
    def calculate_pay(self) -> float:
        """เงินเดือนประจำ"""
        return self.monthly_salary
    
    def get_tax(self) -> float:
        """ภาษี 15% ของเงินเดือน"""
        return self.monthly_salary * self.TAX_RATE
    
    def get_annual_bonus(self) -> float:
        """โบนัสประจำปี 2 เดือน"""
        return self.monthly_salary * 2


class PartTimeEmployee(StaffMember):
    """พนักงานพาร์ทไทม์"""
    TAX_RATE = 0.10  # ภาษี 10%
    
    def __init__(self, employee_id: str, first_name: str, last_name: str,
                 birth_date: date, department: str, 
                 hourly_rate: float, hours_per_month: int):
        super().__init__(employee_id, first_name, last_name, birth_date, department)
        self.hourly_rate = hourly_rate
        self.hours_per_month = hours_per_month
    
    def calculate_pay(self) -> float:
        """ค่าจ้างรายชั่วโมง"""
        return self.hourly_rate * self.hours_per_month
    
    def get_tax(self) -> float:
        """ภาษี 10%"""
        return self.calculate_pay() * self.TAX_RATE


class Contractor(StaffMember):
    """ผู้รับเหมา"""
    TAX_RATE = 0.03  # ภาษีหัก ณ ที่จ่าย 3%
    
    def __init__(self, employee_id: str, first_name: str, last_name: str,
                 birth_date: date, department: str,
                 project_fee: float, contract_months: int):
        super().__init__(employee_id, first_name, last_name, birth_date, department)
        self.project_fee = project_fee
        self.contract_months = contract_months
    
    def calculate_pay(self) -> float:
        """ค่าจ้างต่อเดือน"""
        return self.project_fee / self.contract_months
    
    def get_tax(self) -> float:
        """หัก ณ ที่จ่าย 3%"""
        return self.calculate_pay() * self.TAX_RATE


class SeniorManager(FullTimeEmployee):
    """ผู้จัดการอาวุโส - สืบทอดจาก FullTimeEmployee"""
    BONUS_MULTIPLIER = 3  # โบนัส 3 เดือน
    
    def __init__(self, employee_id: str, first_name: str, last_name: str,
                 birth_date: date, department: str, monthly_salary: float,
                 managed_departments: List[str]):
        super().__init__(employee_id, first_name, last_name, birth_date, 
                        department, monthly_salary)
        self.managed_departments = managed_departments
    
    def get_annual_bonus(self) -> float:
        """Override: โบนัส 3 เดือน"""
        return self.monthly_salary * self.BONUS_MULTIPLIER
    
    def get_info(self) -> dict:
        """Override: เพิ่มข้อมูลแผนกที่ดูแล"""
        info = super().get_info()
        info["managed_departments"] = self.managed_departments
        info["annual_bonus"] = self.get_annual_bonus()
        return info


# ทดสอบระบบพนักงาน
employees = [
    FullTimeEmployee("E001", "สมชาย", "ใจดี", 
                    date(1990, 5, 15), "Engineering", 50000),
    PartTimeEmployee("P001", "สมหญิง", "สวยงาม",
                    date(1995, 8, 20), "Marketing", 200, 80),
    Contractor("C001", "John", "Smith",
              date(1988, 3, 10), "IT", 120000, 6),
    SeniorManager("M001", "นายใหญ่", "บริหารดี",
                 date(1975, 1, 25), "Management", 120000,
                 ["Engineering", "IT", "Marketing"])
]

print("=" * 60)
print("รายงานเงินเดือนพนักงาน")
print("=" * 60)

total_payroll = 0
for emp in employees:
    info = emp.get_info()
    print(f"\n{emp}")
    print(f"  เงินเดือน: {info['gross_pay']:>10,.2f} บาท")
    print(f"  ภาษี:      {info['tax']:>10,.2f} บาท")
    print(f"  รับจริง:   {info['net_pay']:>10,.2f} บาท")
    total_payroll += info['net_pay']

print(f"\n{'='*60}")
print(f"ยอดรวมเงินเดือนสุทธิ: {total_payroll:,.2f} บาท")
```

---

## Exercises

### Exercise 1: Vehicle Hierarchy
สร้างระบบยานพาหนะที่มี:
- `Vehicle` (abstract): `start()`, `stop()`, `fuel_type()`
- `LandVehicle(Vehicle)`: `drive()`
- `WaterVehicle(Vehicle)`: `sail()`
- `Car(LandVehicle)`: ราคา 500,000+ บาท
- `Motorcycle(LandVehicle)`: ราคา 80,000+ บาท
- `Boat(WaterVehicle)`: ราคา 1,000,000+ บาท
- `AmphibiousCar(LandVehicle, WaterVehicle)`: ทั้งขับและแล่นน้ำได้

```python
# แนวทางการแก้ปัญหา
from abc import ABC, abstractmethod

class Vehicle(ABC):
    @abstractmethod
    def start(self) -> str:
        pass
    
    @abstractmethod
    def stop(self) -> str:
        pass
    
    @property
    @abstractmethod
    def fuel_type(self) -> str:
        pass

# TODO: สร้างคลาสที่เหลือ
```

### Exercise 2: Game Character System
สร้างระบบตัวละครเกมที่มี:
- `Character` (base): `name`, `hp`, `mp`, `level`
- `AttackMixin`: `attack()`, `critical_hit()`
- `MagicMixin`: `cast_spell()`, `restore_mp()`
- `HealMixin`: `heal()`, `revive()`
- `Warrior(Character, AttackMixin)`: นักรบ
- `Mage(Character, MagicMixin)`: นักเวทย์
- `Paladin(Character, AttackMixin, HealMixin)`: นักรบศักดิ์สิทธิ์
- `Archmage(Character, MagicMixin, AttackMixin)`: อาร์คเมจ

### Exercise 3: Plugin System
สร้างระบบ plugin โดยใช้ ABC:
- `Plugin` (ABC): `name`, `version`, `execute(data)`
- `DataPlugin(Plugin)`: ประมวลผลข้อมูล
- `ReportPlugin(Plugin)`: สร้างรายงาน
- `PluginManager`: โหลดและรัน plugin

---

## สรุป

| Concept | การใช้งาน | ตัวอย่าง |
|---------|-----------|---------|
| Single Inheritance | สืบทอดจากคลาสเดียว | `class Dog(Animal)` |
| Multiple Inheritance | สืบทอดจากหลายคลาส | `class Duck(Fly, Swim)` |
| `super()` | เรียกเมธอดของ parent | `super().__init__()` |
| MRO | ลำดับการค้นหาเมธอด | `Class.__mro__` |
| ABC | คลาสนามธรรม | `class Base(ABC)` |
| `@abstractmethod` | บังคับ implement | ต้อง override ใน subclass |
| Mixin | เพิ่ม functionality | `class Loggable:` |

**Key Principles:**
- ใช้ inheritance เมื่อ "is-a" relationship เช่น Dog is an Animal
- ใช้ composition เมื่อ "has-a" relationship เช่น Car has an Engine
- ใช้ ABC เพื่อกำหนด interface
- ใช้ Mixin เพื่อ reuse code โดยไม่สร้าง deep hierarchy

---

## ต่อไป

[Part 017 - OOP Polymorphism](part-017.md) - Duck typing, operator overloading, protocols
