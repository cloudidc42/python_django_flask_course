# Part 015: OOP Basics
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- สร้าง Class และ Object ได้
- ใช้ `__init__` กำหนด attributes ได้
- เขียน instance methods, class methods, static methods ได้
- ใช้ properties และ encapsulation ได้
- เข้าใจ `self` ได้อย่างถ่องแท้
- เขียน `__str__` และ `__repr__` ได้

---

## 1. Class และ Object พื้นฐาน

OOP (Object-Oriented Programming) คือการเขียนโปรแกรมที่จัดระเบียบโค้ดเป็น "objects" ที่มี data และ behavior

```python
# ===== Class พื้นฐาน =====

class Dog:
    """แทน dog object"""
    
    # Class attribute - shared โดยทุก instances
    species = "Canis lupus familiaris"
    
    # __init__ - constructor, ทำงานเมื่อสร้าง object
    def __init__(self, name: str, breed: str, age: int):
        # Instance attributes - unique สำหรับแต่ละ object
        self.name = name
        self.breed = breed
        self.age = age
    
    # Instance method
    def bark(self) -> str:
        return f"{self.name}: โฮ่ง โฮ่ง!"
    
    def describe(self) -> str:
        return f"{self.name} เป็น {self.breed} อายุ {self.age} ปี"

# ===== สร้าง Objects (instances) =====
dog1 = Dog("บัดดี้", "Golden Retriever", 3)
dog2 = Dog("แม็กซ์", "Poodle", 2)

# เข้าถึง attributes
print(dog1.name)       # บัดดี้
print(dog2.breed)      # Poodle
print(Dog.species)     # class attribute
print(dog1.species)    # เข้าถึงผ่าน instance ได้ด้วย

# เรียก methods
print(dog1.bark())     # บัดดี้: โฮ่ง โฮ่ง!
print(dog2.describe()) # แม็กซ์ เป็น Poodle อายุ 2 ปี

# แก้ไข attribute
dog1.age = 4           # อัพเดต age
print(dog1.describe()) # อายุ 4 ปีแล้ว

# ตรวจสอบ type
print(f"type: {type(dog1)}")            # <class '__main__.Dog'>
print(f"isinstance: {isinstance(dog1, Dog)}")  # True
```

## 2. self - เข้าใจให้ถ่องแท้

```python
"""
self คือ reference ไปยัง instance ปัจจุบัน
Python ส่งมาให้อัตโนมัติเมื่อเรียก instance method
"""

class Counter:
    def __init__(self, start: int = 0):
        self.count = start    # self.count = instance attribute
    
    def increment(self, amount: int = 1):
        self.count += amount  # แก้ไข attribute ของ instance นี้
        return self           # return self เพื่อทำ method chaining
    
    def reset(self):
        self.count = 0
        return self
    
    def get(self) -> int:
        return self.count

c1 = Counter(10)
c2 = Counter(0)

c1.increment(5)
c2.increment(3)

print(f"c1: {c1.get()}")  # 15
print(f"c2: {c2.get()}")  # 3

# ===== Method Chaining =====
# เพราะ return self ทุก method
result = Counter(0).increment(10).increment(5).increment(3).get()
print(f"Chained: {result}")  # 18

# ===== self คือ instance เอง =====
class Printer:
    def print_self(self):
        print(f"self is: {self}")
        print(f"self id: {id(self)}")

p = Printer()
print(f"p id: {id(p)}")
p.print_self()  # id จะตรงกัน

# ===== เรียก method เองผ่าน self =====
class BankAccount:
    def __init__(self, owner: str, balance: float = 0):
        self.owner = owner
        self._balance = balance
        self._transactions = []
    
    def _log(self, action: str, amount: float) -> None:
        """Private method สำหรับบันทึก transaction"""
        self._transactions.append({
            "action": action,
            "amount": amount,
            "balance": self._balance
        })
    
    def deposit(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("ต้องฝากมากกว่า 0")
        self._balance += amount
        self._log("deposit", amount)  # เรียก method อื่นผ่าน self
        print(f"ฝาก {amount:,.2f} บาท, ยอดคงเหลือ: {self._balance:,.2f}")
    
    def withdraw(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("ต้องถอนมากกว่า 0")
        if amount > self._balance:
            raise ValueError("ยอดเงินไม่พอ")
        self._balance -= amount
        self._log("withdraw", amount)
        print(f"ถอน {amount:,.2f} บาท, ยอดคงเหลือ: {self._balance:,.2f}")
    
    def show_history(self) -> None:
        print(f"\nประวัติ {self.owner}:")
        for t in self._transactions:
            print(f"  {t['action']:10} {t['amount']:>10,.2f}  → {t['balance']:>10,.2f}")

acc = BankAccount("สมชาย", 1000)
acc.deposit(500)
acc.deposit(2000)
acc.withdraw(750)
acc.show_history()
```

## 3. Instance Methods

```python
class Circle:
    """แทน circle shape"""
    
    PI = 3.14159265358979  # class constant
    
    def __init__(self, radius: float):
        if radius <= 0:
            raise ValueError(f"radius ต้องมากกว่า 0 (ได้รับ: {radius})")
        self.radius = radius
    
    # ===== Instance methods =====
    def area(self) -> float:
        """คำนวณพื้นที่"""
        return self.PI * self.radius ** 2
    
    def circumference(self) -> float:
        """คำนวณเส้นรอบวง"""
        return 2 * self.PI * self.radius
    
    def scale(self, factor: float) -> "Circle":
        """สร้าง Circle ใหม่ที่ขยาย/หดส่วน"""
        return Circle(self.radius * factor)
    
    def is_larger_than(self, other: "Circle") -> bool:
        """เปรียบเทียบกับ Circle อื่น"""
        return self.area() > other.area()
    
    def __str__(self) -> str:
        return f"Circle(r={self.radius:.2f})"
    
    def __repr__(self) -> str:
        return f"Circle(radius={self.radius!r})"

c1 = Circle(5)
c2 = Circle(3)
c3 = c1.scale(2)

print(f"Circle: {c1}")
print(f"Area: {c1.area():.2f}")
print(f"Circumference: {c1.circumference():.2f}")
print(f"c1 > c2: {c1.is_larger_than(c2)}")
print(f"Scaled: {c3}")
```

## 4. Class Methods

```python
from datetime import date

class Person:
    """แทน person"""
    
    _count = 0  # นับ instances
    
    def __init__(self, name: str, birthdate: date):
        self.name = name
        self.birthdate = birthdate
        Person._count += 1
    
    @property
    def age(self) -> int:
        """คำนวณอายุ"""
        today = date.today()
        years = today.year - self.birthdate.year
        if (today.month, today.day) < (self.birthdate.month, self.birthdate.day):
            years -= 1
        return years
    
    # ===== Class Methods =====
    @classmethod
    def from_birth_year(cls, name: str, birth_year: int) -> "Person":
        """Alternative constructor จากปีเกิด"""
        birthdate = date(birth_year, 1, 1)
        return cls(name, birthdate)
    
    @classmethod
    def from_string(cls, data: str) -> "Person":
        """Alternative constructor จาก string 'name,YYYY-MM-DD'"""
        name, date_str = data.split(",")
        year, month, day = map(int, date_str.strip().split("-"))
        return cls(name.strip(), date(year, month, day))
    
    @classmethod
    def get_count(cls) -> int:
        """จำนวน Person instances ทั้งหมด"""
        return cls._count
    
    def __str__(self) -> str:
        return f"{self.name} (อายุ {self.age} ปี)"

# สร้างด้วยวิธีปกติ
p1 = Person("สมชาย", date(1993, 5, 15))
print(p1)

# สร้างด้วย class methods (alternative constructors)
p2 = Person.from_birth_year("สมหญิง", 1998)
print(p2)

p3 = Person.from_string("สมศักดิ์, 1985-08-22")
print(p3)

print(f"จำนวน Person ทั้งหมด: {Person.get_count()}")
```

## 5. Static Methods

```python
class MathUtils:
    """Utility class สำหรับ math operations"""
    
    @staticmethod
    def is_prime(n: int) -> bool:
        """ตรวจสอบว่าเป็นเลขเฉพาะไหม"""
        if n < 2:
            return False
        if n == 2:
            return True
        if n % 2 == 0:
            return False
        for i in range(3, int(n**0.5) + 1, 2):
            if n % i == 0:
                return False
        return True
    
    @staticmethod
    def fibonacci(n: int) -> list:
        """สร้าง Fibonacci sequence n ตัว"""
        if n <= 0:
            return []
        if n == 1:
            return [0]
        fib = [0, 1]
        for _ in range(2, n):
            fib.append(fib[-1] + fib[-2])
        return fib
    
    @staticmethod
    def gcd(a: int, b: int) -> int:
        """หา Greatest Common Divisor"""
        while b:
            a, b = b, a % b
        return a
    
    @staticmethod
    def lcm(a: int, b: int) -> int:
        """หา Least Common Multiple"""
        return abs(a * b) // MathUtils.gcd(a, b)

# เรียกใช้ผ่าน class (ไม่ต้องสร้าง instance)
print(f"7 เป็นเลขเฉพาะ: {MathUtils.is_prime(7)}")
print(f"10 เป็นเลขเฉพาะ: {MathUtils.is_prime(10)}")
print(f"Fibonacci 10 ตัว: {MathUtils.fibonacci(10)}")
print(f"GCD(12, 8): {MathUtils.gcd(12, 8)}")
print(f"LCM(4, 6): {MathUtils.lcm(4, 6)}")

# เรียกผ่าน instance ก็ได้ (แต่ไม่จำเป็น)
utils = MathUtils()
primes = [n for n in range(2, 50) if utils.is_prime(n)]
print(f"เลขเฉพาะ 2-50: {primes}")

# ===== เปรียบเทียบ =====
"""
Instance method:  self เป็น argument แรก, เข้าถึง instance data ได้
Class method:     cls เป็น argument แรก, เข้าถึง class data ได้
Static method:    ไม่มี special argument, เหมือน function ธรรมดา
                  แต่อยู่ใน namespace ของ class
"""
```

## 6. Properties และ Encapsulation

```python
class Temperature:
    """แปลงหน่วยอุณหภูมิ"""
    
    def __init__(self, celsius: float = 0):
        self._celsius = celsius  # _name = convention สำหรับ protected
    
    # ===== Property - getter =====
    @property
    def celsius(self) -> float:
        """อุณหภูมิเซลเซียส"""
        return self._celsius
    
    # ===== Property - setter =====
    @celsius.setter
    def celsius(self, value: float) -> None:
        if value < -273.15:
            raise ValueError(f"ต่ำกว่า absolute zero ไม่ได้ (ได้รับ: {value})")
        self._celsius = value
    
    # ===== Computed properties (read-only) =====
    @property
    def fahrenheit(self) -> float:
        return (self._celsius * 9/5) + 32
    
    @property
    def kelvin(self) -> float:
        return self._celsius + 273.15
    
    @fahrenheit.setter
    def fahrenheit(self, value: float) -> None:
        self.celsius = (value - 32) * 5/9  # ใช้ setter ของ celsius
    
    def __str__(self) -> str:
        return (f"{self._celsius:.2f}°C = "
                f"{self.fahrenheit:.2f}°F = "
                f"{self.kelvin:.2f}K")

temp = Temperature(25)
print(temp)  # 25.00°C = 77.00°F = 298.15K

temp.celsius = 100
print(f"100°C = {temp.fahrenheit}°F")  # 212.0

temp.fahrenheit = 32
print(f"32°F = {temp.celsius}°C")  # 0.0

try:
    temp.celsius = -300  # ต่ำกว่า absolute zero
except ValueError as e:
    print(f"Error: {e}")

# ===== Encapsulation ใน Class ที่ซับซ้อนขึ้น =====
class Employee:
    """พนักงาน - แสดง encapsulation"""
    
    def __init__(self, employee_id: str, name: str, salary: float):
        self.employee_id = employee_id
        self.name = name
        self._salary = salary        # protected
        self.__tax_rate = 0.15       # private (name mangling)
        self._raise_history = []
    
    @property
    def salary(self) -> float:
        return self._salary
    
    @salary.setter
    def salary(self, value: float) -> None:
        if value < 0:
            raise ValueError("เงินเดือนต้องไม่ติดลบ")
        old_salary = self._salary
        self._salary = value
        if old_salary > 0:
            change = ((value - old_salary) / old_salary) * 100
            self._raise_history.append({
                "old": old_salary,
                "new": value,
                "change_pct": round(change, 2)
            })
    
    @property
    def net_salary(self) -> float:
        """เงินเดือนหลังหักภาษี"""
        return self._salary * (1 - self.__tax_rate)
    
    @property
    def raise_history(self) -> list:
        return self._raise_history.copy()  # return copy เพื่อความปลอดภัย
    
    def give_raise(self, percentage: float) -> None:
        """ขึ้นเงินเดือน"""
        new_salary = self._salary * (1 + percentage / 100)
        self.salary = new_salary
        print(f"ขึ้นเงินเดือน {self.name}: {self._salary:,.2f} บาท (+{percentage}%)")
    
    def __str__(self) -> str:
        return f"Employee({self.employee_id}: {self.name}, {self._salary:,.2f} บาท)"

emp = Employee("E001", "สมชาย ใจดี", 35000)
print(emp)
print(f"Net salary: {emp.net_salary:,.2f} บาท")

emp.give_raise(10)
emp.give_raise(15)

print("\nประวัติการขึ้นเงินเดือน:")
for raise_record in emp.raise_history:
    print(f"  {raise_record['old']:,.2f} → {raise_record['new']:,.2f} "
          f"({raise_record['change_pct']:+.1f}%)")

# ===== Name Mangling =====
print(f"\nPrivate attribute: {emp._Employee__tax_rate}")  # name mangling
```

## 7. `__str__` และ `__repr__`

```python
class Book:
    """หนังสือ"""
    
    def __init__(self, title: str, author: str, isbn: str, price: float):
        self.title = title
        self.author = author
        self.isbn = isbn
        self.price = price
    
    def __str__(self) -> str:
        """สำหรับ end users - readable"""
        return f'"{self.title}" โดย {self.author} ({self.price:,.2f} บาท)'
    
    def __repr__(self) -> str:
        """สำหรับ developers - unambiguous"""
        return (f"Book(title={self.title!r}, "
                f"author={self.author!r}, "
                f"isbn={self.isbn!r}, "
                f"price={self.price})")
    
    def __len__(self) -> int:
        """len(book) = จำนวนตัวอักษรในชื่อ"""
        return len(self.title)
    
    def __eq__(self, other) -> bool:
        """เปรียบเทียบด้วย isbn"""
        if not isinstance(other, Book):
            return False
        return self.isbn == other.isbn
    
    def __lt__(self, other) -> bool:
        """เรียงลำดับตามราคา"""
        if not isinstance(other, Book):
            return NotImplemented
        return self.price < other.price
    
    def __hash__(self):
        """ใช้ใน set/dict"""
        return hash(self.isbn)

b1 = Book("Python Programming", "สมชาย", "978-1234567890", 599)
b2 = Book("Django Web Dev", "สมหญิง", "978-0987654321", 799)
b3 = Book("Python Programming", "สมชาย", "978-1234567890", 599)

# str() สำหรับ users
print(str(b1))      # "Python Programming" โดย สมชาย (599.00 บาท)
print(b1)           # เรียก __str__ อัตโนมัติ

# repr() สำหรับ developers
print(repr(b1))     # Book(title='Python Programming', ...)

# len
print(f"ความยาวชื่อ: {len(b1)}")  # 18

# เปรียบเทียบ
print(f"b1 == b3: {b1 == b3}")  # True (isbn เหมือนกัน)
print(f"b1 == b2: {b1 == b2}")  # False

# เรียงลำดับ
books = [b2, b1, b3]
sorted_books = sorted(books)
for book in sorted_books:
    print(f"  {book.price:,.2f}: {book.title}")

# ใช้ใน set (ต้องมี __hash__)
book_set = {b1, b2, b3}  # b1 และ b3 ซ้ำกัน
print(f"Unique books: {len(book_set)}")  # 2
```

## 8. Dunder Methods เพิ่มเติม

```python
class Vector:
    """Vector คณิตศาสตร์"""
    
    def __init__(self, x: float, y: float, z: float = 0):
        self.x = x
        self.y = y
        self.z = z
    
    def __str__(self) -> str:
        return f"Vector({self.x}, {self.y}, {self.z})"
    
    def __repr__(self) -> str:
        return f"Vector({self.x!r}, {self.y!r}, {self.z!r})"
    
    # Arithmetic
    def __add__(self, other: "Vector") -> "Vector":
        return Vector(self.x + other.x, self.y + other.y, self.z + other.z)
    
    def __sub__(self, other: "Vector") -> "Vector":
        return Vector(self.x - other.x, self.y - other.y, self.z - other.z)
    
    def __mul__(self, scalar: float) -> "Vector":
        return Vector(self.x * scalar, self.y * scalar, self.z * scalar)
    
    def __rmul__(self, scalar: float) -> "Vector":
        return self.__mul__(scalar)  # 3 * v = v * 3
    
    def __neg__(self) -> "Vector":
        return Vector(-self.x, -self.y, -self.z)
    
    def __abs__(self) -> float:
        """magnitude ของ vector"""
        return (self.x**2 + self.y**2 + self.z**2) ** 0.5
    
    def __eq__(self, other) -> bool:
        if not isinstance(other, Vector):
            return False
        return (self.x == other.x and self.y == other.y and self.z == other.z)
    
    def __bool__(self) -> bool:
        """Vector เป็น False ถ้าเป็น zero vector"""
        return bool(self.x or self.y or self.z)
    
    def dot(self, other: "Vector") -> float:
        """Dot product"""
        return self.x*other.x + self.y*other.y + self.z*other.z
    
    def normalize(self) -> "Vector":
        """Unit vector"""
        mag = abs(self)
        if mag == 0:
            raise ValueError("Zero vector ไม่มี unit vector")
        return Vector(self.x/mag, self.y/mag, self.z/mag)

v1 = Vector(1, 2, 3)
v2 = Vector(4, 5, 6)

print(f"v1 = {v1}")
print(f"v2 = {v2}")
print(f"v1 + v2 = {v1 + v2}")
print(f"v1 - v2 = {v1 - v2}")
print(f"v1 * 3 = {v1 * 3}")
print(f"3 * v1 = {3 * v1}")
print(f"-v1 = {-v1}")
print(f"|v1| = {abs(v1):.4f}")
print(f"v1 · v2 = {v1.dot(v2)}")
print(f"unit v1 = {v1.normalize()}")
print(f"v1 == v1: {v1 == Vector(1, 2, 3)}")
print(f"bool(v1): {bool(v1)}")
print(f"bool(Vector(0,0,0)): {bool(Vector(0,0,0))}")
```

## 9. ตัวอย่างโปรเจกต์: ระบบจัดการร้านค้า

```python
"""
ระบบร้านค้า แสดงหลักการ OOP ครบถ้วน
"""

from datetime import datetime
from typing import Optional

class Product:
    """สินค้า"""
    
    def __init__(self, product_id: str, name: str, price: float, 
                 stock: int = 0, category: str = "general"):
        self.product_id = product_id
        self.name = name
        self._price = price
        self._stock = stock
        self.category = category
    
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
    
    def add_stock(self, quantity: int) -> None:
        if quantity <= 0:
            raise ValueError("จำนวนต้องมากกว่า 0")
        self._stock += quantity
    
    def reduce_stock(self, quantity: int) -> bool:
        if quantity <= 0:
            raise ValueError("จำนวนต้องมากกว่า 0")
        if quantity > self._stock:
            return False
        self._stock -= quantity
        return True
    
    def __str__(self) -> str:
        status = "มีสินค้า" if self.in_stock else "หมด"
        return f"{self.product_id}: {self.name} ({self._price:,.2f}฿) [{status}]"
    
    def __repr__(self) -> str:
        return (f"Product(id={self.product_id!r}, name={self.name!r}, "
                f"price={self._price}, stock={self._stock})")


class CartItem:
    """สินค้าใน cart"""
    
    def __init__(self, product: Product, quantity: int):
        self.product = product
        self.quantity = quantity
    
    @property
    def subtotal(self) -> float:
        return self.product.price * self.quantity
    
    def __str__(self) -> str:
        return (f"{self.product.name} x{self.quantity} "
                f"= {self.subtotal:,.2f}฿")


class ShoppingCart:
    """ตะกร้าสินค้า"""
    
    def __init__(self):
        self._items: dict = {}  # {product_id: CartItem}
    
    def add_item(self, product: Product, quantity: int = 1) -> None:
        """เพิ่มสินค้าลงตะกร้า"""
        if not product.in_stock:
            raise ValueError(f"สินค้า {product.name} หมดแล้ว")
        if quantity > product.stock:
            raise ValueError(f"สต็อกไม่พอ (มี {product.stock} ชิ้น)")
        
        if product.product_id in self._items:
            self._items[product.product_id].quantity += quantity
        else:
            self._items[product.product_id] = CartItem(product, quantity)
    
    def remove_item(self, product_id: str) -> bool:
        """ลบสินค้าออกจากตะกร้า"""
        if product_id in self._items:
            del self._items[product_id]
            return True
        return False
    
    def update_quantity(self, product_id: str, quantity: int) -> bool:
        """อัพเดตจำนวน"""
        if product_id not in self._items:
            return False
        if quantity <= 0:
            return self.remove_item(product_id)
        self._items[product_id].quantity = quantity
        return True
    
    @property
    def items(self) -> list:
        return list(self._items.values())
    
    @property
    def total(self) -> float:
        return sum(item.subtotal for item in self._items.values())
    
    @property
    def item_count(self) -> int:
        return sum(item.quantity for item in self._items.values())
    
    def is_empty(self) -> bool:
        return len(self._items) == 0
    
    def clear(self) -> None:
        self._items.clear()
    
    def show(self) -> None:
        print("\n🛒 ตะกร้าสินค้า:")
        print("-" * 50)
        if self.is_empty():
            print("  ตะกร้าว่างเปล่า")
        else:
            for item in self.items:
                print(f"  {item}")
            print("-" * 50)
            print(f"  รวม: {self.total:,.2f} บาท ({self.item_count} ชิ้น)")


class Order:
    """คำสั่งซื้อ"""
    
    _order_counter = 0
    
    def __init__(self, cart: ShoppingCart, customer_name: str):
        Order._order_counter += 1
        self.order_id = f"ORD{Order._order_counter:05d}"
        self.customer_name = customer_name
        self.items = cart.items.copy()
        self.total = cart.total
        self.timestamp = datetime.now()
        self.status = "pending"
    
    @classmethod
    def get_order_count(cls) -> int:
        return cls._order_counter
    
    @staticmethod
    def calculate_shipping(total: float) -> float:
        """คำนวณค่าส่ง"""
        if total >= 500:
            return 0  # ฟรีค่าส่ง
        return 50
    
    @property
    def shipping_fee(self) -> float:
        return self.calculate_shipping(self.total)
    
    @property
    def grand_total(self) -> float:
        return self.total + self.shipping_fee
    
    def confirm(self) -> None:
        """ยืนยันคำสั่งซื้อ"""
        for item in self.items:
            success = item.product.reduce_stock(item.quantity)
            if not success:
                raise ValueError(f"สต็อก {item.product.name} ไม่พอ")
        self.status = "confirmed"
    
    def show_receipt(self) -> None:
        print(f"\n{'='*50}")
        print(f"  ใบเสร็จ #{self.order_id}")
        print(f"  ลูกค้า: {self.customer_name}")
        print(f"  วันที่: {self.timestamp.strftime('%d/%m/%Y %H:%M')}")
        print(f"{'='*50}")
        for item in self.items:
            print(f"  {item.product.name:25} {item.quantity:3} x {item.product.price:8,.2f} = {item.subtotal:10,.2f}")
        print(f"{'='*50}")
        print(f"  {'ราคาสินค้า':35} {self.total:>10,.2f}")
        if self.shipping_fee > 0:
            print(f"  {'ค่าจัดส่ง':35} {self.shipping_fee:>10,.2f}")
        else:
            print(f"  {'ค่าจัดส่ง (ฟรี เมื่อสั่งครบ 500฿)':35} {'0.00':>10}")
        print(f"{'='*50}")
        print(f"  {'ยอดรวมทั้งหมด':35} {self.grand_total:>10,.2f}")
        print(f"{'='*50}")
        print(f"  สถานะ: {self.status}")


# ===== ทดสอบระบบ =====

# สร้างสินค้า
products = [
    Product("P001", "Python Programming Book", 599, 20, "books"),
    Product("P002", "USB-C Hub", 890, 15, "electronics"),
    Product("P003", "Mechanical Keyboard", 3500, 5, "electronics"),
    Product("P004", "Notebook (50 แผ่น)", 89, 100, "stationery"),
    Product("P005", "Pen Set (12 ด้าม)", 149, 50, "stationery"),
]

print("สินค้าในร้าน:")
for p in products:
    print(f"  {p}")

# สร้าง cart
cart = ShoppingCart()
cart.add_item(products[0], 2)   # Python Book x2
cart.add_item(products[3], 5)   # Notebook x5
cart.add_item(products[4], 3)   # Pen Set x3

cart.show()

# สั่งซื้อ
order = Order(cart, "สมชาย ใจดี")
order.confirm()
order.show_receipt()

# สั่งซื้ออีกครั้ง
cart2 = ShoppingCart()
cart2.add_item(products[2], 1)  # Keyboard x1

order2 = Order(cart2, "สมหญิง สวยงาม")
order2.confirm()
order2.show_receipt()

print(f"\nจำนวน orders ทั้งหมด: {Order.get_order_count()}")
print(f"\nสต็อกคงเหลือ:")
for p in products:
    print(f"  {p.name}: {p.stock} ชิ้น")
```

## 10. สรุป Part 015

ใน Part นี้คุณได้เรียนรู้:

✅ **Class และ Object** - สร้าง class, instantiate objects, class attributes vs instance attributes
✅ **`__init__`** - constructor สำหรับกำหนด initial state
✅ **self** - reference ไปยัง instance ปัจจุบัน, method chaining
✅ **Instance Methods** - methods ที่ทำงานกับ instance data
✅ **Class Methods** - `@classmethod`, alternative constructors, class-level operations
✅ **Static Methods** - `@staticmethod`, utility functions ใน namespace ของ class
✅ **Properties** - `@property`, getter/setter, computed properties
✅ **Encapsulation** - `_protected`, `__private`, name mangling
✅ **`__str__` และ `__repr__`** - สำหรับ users vs developers
✅ **Dunder Methods** - `__add__`, `__eq__`, `__len__`, `__bool__` และอื่นๆ

---

## ➡️ ถัดไป: Part 016 - OOP Advanced (Inheritance, Polymorphism)

*Part 015/100+ | Python Course - Beginner to World-Class*
