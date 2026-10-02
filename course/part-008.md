# Part 008: Tuples
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- สร้างและใช้งาน Tuple ได้
- เข้าใจความหมายของ Immutability
- ใช้ Tuple Packing และ Unpacking ได้
- สร้างและใช้ Named Tuples ได้
- เลือกใช้ Tuple หรือ List ได้อย่างเหมาะสม

---

## 1. Tuple พื้นฐาน

```python
# สร้าง Tuple ด้วยวิธีต่างๆ
empty_tuple = ()                         # tuple ว่าง
single = (42,)                           # tuple 1 ตัว (ต้องมี comma!)
point = (3, 4)                           # tuple 2 ตัว
coords = (10.5, 20.3, 5.0)              # tuple 3 ตัว
mixed = (1, "hello", 3.14, True)         # tuple ผสม type
nested = ((1, 2), (3, 4), (5, 6))       # nested tuple

# สร้างด้วย tuple() constructor
from_list = tuple([1, 2, 3, 4, 5])
from_range = tuple(range(1, 6))
from_string = tuple("Python")

print(from_list)    # (1, 2, 3, 4, 5)
print(from_range)   # (1, 2, 3, 4, 5)
print(from_string)  # ('P', 'y', 't', 'h', 'o', 'n')

# ⚠️ Tuple 1 ตัวต้องมี trailing comma
not_tuple = (42)    # นี่คือ int ธรรมดา!
real_tuple = (42,)  # นี่คือ tuple 1 ตัว

print(type(not_tuple))   # <class 'int'>
print(type(real_tuple))  # <class 'tuple'>

# Tuple Packing (ไม่ต้องใส่วงเล็บก็ได้)
coordinates = 10, 20, 30
print(coordinates)        # (10, 20, 30)
print(type(coordinates))  # <class 'tuple'>

# การเข้าถึงสมาชิก (เหมือน list)
colors = ("red", "green", "blue", "yellow", "purple")

print(colors[0])    # red
print(colors[-1])   # purple
print(colors[1:3])  # ('green', 'blue')
print(len(colors))  # 5

# ตรวจสอบสมาชิก
print("red" in colors)     # True
print("orange" in colors)  # False

# Iteration
for color in colors:
    print(color, end=" ")
print()  # red green blue yellow purple

# count() และ index()
nums = (1, 2, 3, 2, 1, 4, 2)
print(nums.count(2))   # 3
print(nums.index(3))   # 2
```

---

## 2. Immutability (ความไม่เปลี่ยนแปลง)

```python
# Tuple ไม่สามารถแก้ไขได้หลังสร้าง
point = (3, 4)

# ลองแก้ไข - จะได้ TypeError
try:
    point[0] = 10
except TypeError as e:
    print(f"Error: {e}")  # 'tuple' object does not support item assignment

# ลองลบ - จะได้ TypeError
try:
    del point[0]
except TypeError as e:
    print(f"Error: {e}")  # 'tuple' object doesn't support item deletion

# ลอง append - จะได้ AttributeError
try:
    point.append(5)
except AttributeError as e:
    print(f"Error: {e}")  # 'tuple' object has no attribute 'append'

# แต่ถ้า tuple มี mutable object ข้างใน...
mutable_in_tuple = ([1, 2, 3], [4, 5, 6])
print(mutable_in_tuple)  # ([1, 2, 3], [4, 5, 6])

# ไม่สามารถเปลี่ยน reference ได้
try:
    mutable_in_tuple[0] = [7, 8, 9]
except TypeError:
    print("ไม่สามารถเปลี่ยน reference ได้")

# แต่สามารถแก้ไข list ข้างในได้!
mutable_in_tuple[0].append(99)
print(mutable_in_tuple)  # ([1, 2, 3, 99], [4, 5, 6])  ← list ข้างในเปลี่ยนได้

# Immutability ทำให้ tuple เป็น hashable (ใช้เป็น dict key ได้)
locations = {
    (0, 0): "origin",
    (1, 0): "right",
    (0, 1): "up",
    (-1, 0): "left",
}
print(locations[(1, 0)])  # right

# list ใช้เป็น dict key ไม่ได้
try:
    bad_dict = {[1, 2]: "value"}
except TypeError as e:
    print(f"Error: {e}")  # unhashable type: 'list'
```

---

## 3. Tuple Unpacking

```python
# Basic unpacking
point = (10, 20)
x, y = point
print(f"x={x}, y={y}")  # x=10, y=20

# 3D coordinates
x, y, z = (1.5, 2.7, 3.2)
print(f"x={x}, y={y}, z={z}")

# แลกค่าตัวแปร (swap) - Pythonic way
a, b = 5, 10
print(f"before: a={a}, b={b}")
a, b = b, a                    # swap โดยใช้ tuple unpacking
print(f"after: a={a}, b={b}")  # after: a=10, b=5

# Extended unpacking ด้วย *
first, *rest = (1, 2, 3, 4, 5)
print(first)  # 1
print(rest)   # [2, 3, 4, 5]

*init, last = (1, 2, 3, 4, 5)
print(init)   # [1, 2, 3, 4]
print(last)   # 5

first, *middle, last = (1, 2, 3, 4, 5)
print(first)   # 1
print(middle)  # [2, 3, 4]
print(last)    # 5

# Ignore ค่าที่ไม่ต้องการด้วย _
name, _, age = ("Alice", "Developer", 30)
print(f"{name} อายุ {age} ปี")

# Nested unpacking
(a, b), (c, d) = (1, 2), (3, 4)
print(a, b, c, d)  # 1 2 3 4

matrix = ((1, 2, 3), (4, 5, 6), (7, 8, 9))
(r1c1, r1c2, r1c3), _, (r3c1, r3c2, r3c3) = matrix
print(r1c1, r3c3)  # 1 9

# Unpacking ใน for loop
students = [("Alice", 85), ("Bob", 92), ("Charlie", 78)]
for name, score in students:
    print(f"{name}: {score}")
# Alice: 85
# Bob: 92
# Charlie: 78

# Unpacking dict items
person = {"name": "Alice", "age": 30, "city": "Bangkok"}
for key, value in person.items():
    print(f"{key}: {value}")
```

---

## 4. Tuple เป็น Return Value

```python
# Tuple ใช้ return หลายค่าจากฟังก์ชัน
def min_max(numbers):
    """คืน tuple ของ (min, max)"""
    return min(numbers), max(numbers)

data = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3]
minimum, maximum = min_max(data)
print(f"min={minimum}, max={maximum}")  # min=1, max=9

def divmod_custom(dividend, divisor):
    """คืน quotient และ remainder"""
    quotient = dividend // divisor
    remainder = dividend % divisor
    return quotient, remainder

q, r = divmod_custom(17, 5)
print(f"17 ÷ 5 = {q} เศษ {r}")  # 17 ÷ 5 = 3 เศษ 2

# Python built-in divmod()
q, r = divmod(17, 5)
print(f"17 ÷ 5 = {q} เศษ {r}")  # เหมือนกัน

# ฟังก์ชันที่คืน tuple ของสถิติ
def describe_list(lst):
    """คืนสถิติพื้นฐาน"""
    n = len(lst)
    total = sum(lst)
    mean = total / n
    sorted_lst = sorted(lst)
    median = sorted_lst[n // 2] if n % 2 == 1 else (sorted_lst[n//2 - 1] + sorted_lst[n//2]) / 2
    return n, total, mean, median, min(lst), max(lst)

n, total, mean, median, lo, hi = describe_list([3, 1, 4, 1, 5, 9, 2, 6])
print(f"จำนวน: {n}, รวม: {total}, เฉลี่ย: {mean:.2f}")
print(f"มัธยฐาน: {median}, ต่ำสุด: {lo}, สูงสุด: {hi}")
```

---

## 5. Named Tuples

```python
from collections import namedtuple

# สร้าง Named Tuple
Point = namedtuple("Point", ["x", "y"])
p1 = Point(3, 4)
p2 = Point(x=10, y=20)

# เข้าถึงด้วยชื่อ (readable)
print(p1.x, p1.y)    # 3 4
print(p2.x, p2.y)    # 10 20

# ยังใช้ index ได้เหมือนเดิม
print(p1[0], p1[1])  # 3 4

# Unpacking ได้
x, y = p1
print(x, y)  # 3 4

# Named Tuple สำหรับ data record
Person = namedtuple("Person", ["name", "age", "email"])
alice = Person("Alice", 30, "alice@example.com")
bob = Person(name="Bob", age=25, email="bob@example.com")

print(alice)         # Person(name='Alice', age=30, email='alice@example.com')
print(alice.name)    # Alice
print(alice.email)   # alice@example.com

# Named Tuple สำหรับ 2D vector
Vector2D = namedtuple("Vector2D", ["x", "y"])

def vector_add(v1, v2):
    return Vector2D(v1.x + v2.x, v1.y + v2.y)

def vector_magnitude(v):
    import math
    return math.sqrt(v.x**2 + v.y**2)

v1 = Vector2D(3, 4)
v2 = Vector2D(1, 2)
v3 = vector_add(v1, v2)

print(f"v1 + v2 = {v3}")                       # v1 + v2 = Vector2D(x=4, y=6)
print(f"|v1| = {vector_magnitude(v1):.2f}")    # |v1| = 5.00

# _replace() - สร้าง tuple ใหม่โดยเปลี่ยนค่าบางส่วน
alice_updated = alice._replace(age=31, email="alice.new@example.com")
print(alice_updated)  # Person(name='Alice', age=31, email='alice.new@example.com')
print(alice)          # Person(name='Alice', age=30, ...) ← ไม่เปลี่ยน

# _asdict() - แปลงเป็น dict
person_dict = alice._asdict()
print(person_dict)   # {'name': 'Alice', 'age': 30, 'email': 'alice@example.com'}
print(type(person_dict))  # <class 'dict'>

# _fields - ดู field names
print(Person._fields)  # ('name', 'age', 'email')

# ใช้ typing.NamedTuple (Python 3.6+) - แนะนำ
from typing import NamedTuple

class Employee(NamedTuple):
    name: str
    department: str
    salary: float
    years: int = 0  # มี default value ได้

emp1 = Employee("Alice", "Engineering", 80000.0, 3)
emp2 = Employee("Bob", "Marketing", 65000.0)

print(emp1)           # Employee(name='Alice', department='Engineering', ...)
print(emp1.salary)    # 80000.0
print(emp2.years)     # 0  ← ใช้ default value

# เรียงตามเงินเดือน
employees = [emp1, emp2, Employee("Charlie", "Engineering", 90000.0, 5)]
top_earner = max(employees, key=lambda e: e.salary)
print(f"เงินเดือนสูงสุด: {top_earner.name} ({top_earner.salary:,.0f} บาท)")
```

---

## 6. Tuple vs List

```python
# เปรียบเทียบ
import sys

lst = [1, 2, 3, 4, 5]
tup = (1, 2, 3, 4, 5)

# Memory usage
print(f"List size: {sys.getsizeof(lst)} bytes")
print(f"Tuple size: {sys.getsizeof(tup)} bytes")
# Tuple ใช้ memory น้อยกว่า

# Speed
import timeit

# Create
list_time = timeit.timeit("[1, 2, 3, 4, 5]", number=1000000)
tuple_time = timeit.timeit("(1, 2, 3, 4, 5)", number=1000000)
print(f"List create: {list_time:.3f}s")
print(f"Tuple create: {tuple_time:.3f}s")

# Access
list_access = timeit.timeit("x[2]", "x = [1,2,3,4,5]", number=1000000)
tuple_access = timeit.timeit("x[2]", "x = (1,2,3,4,5)", number=1000000)
print(f"List access: {list_access:.3f}s")
print(f"Tuple access: {tuple_access:.3f}s")
```

```python
# เมื่อไหรควรใช้ Tuple
# 1. ข้อมูลที่ไม่ควรเปลี่ยน (constants, configuration)
WINDOW_SIZE = (1920, 1080)    # ขนาดหน้าจอ
SERVER_ADDRESS = ("localhost", 8080)

# 2. Return multiple values จากฟังก์ชัน
def get_bounds(data):
    return min(data), max(data)

# 3. Dictionary keys
graph = {
    (0, 0): ["right", "down"],
    (1, 0): ["left", "down"],
    (0, 1): ["up", "right"],
}

# 4. ข้อมูลที่มีโครงสร้างชัดเจน (heterogeneous)
PERSON = ("Alice", 30, "Bangkok")  # (name, age, city) - ใช้ NamedTuple ดีกว่า

# 5. Swap variables
a, b = b, a

# เมื่อไหรควรใช้ List
# 1. ข้อมูลที่ต้องเปลี่ยนแปลง (add/remove)
shopping_cart = ["apple", "banana"]
shopping_cart.append("cherry")

# 2. ข้อมูลที่ homogeneous (ประเภทเดียวกัน)
scores = [85, 92, 78, 95, 88]

# 3. ต้องการ sort/filter
scores.sort()
passing = [s for s in scores if s >= 70]
```

---

## 7. Use Cases จริง

```python
# 1. Database records
from collections import namedtuple

Record = namedtuple("Record", ["id", "name", "value", "timestamp"])
records = [
    Record(1, "temperature", 25.5, "2024-01-01 08:00"),
    Record(2, "humidity", 65.0, "2024-01-01 08:00"),
    Record(3, "pressure", 1013.25, "2024-01-01 08:00"),
]

for rec in records:
    print(f"[{rec.id}] {rec.name}: {rec.value} @ {rec.timestamp}")

# 2. Coordinate system
from typing import NamedTuple
import math

class Point3D(NamedTuple):
    x: float
    y: float
    z: float
    
    def distance_to(self, other: "Point3D") -> float:
        return math.sqrt(
            (self.x - other.x)**2 +
            (self.y - other.y)**2 +
            (self.z - other.z)**2
        )
    
    def __str__(self):
        return f"({self.x}, {self.y}, {self.z})"

p1 = Point3D(0, 0, 0)
p2 = Point3D(3, 4, 0)
p3 = Point3D(1, 1, 1)

print(f"p1 = {p1}")
print(f"p2 = {p2}")
print(f"ระยะ p1→p2 = {p1.distance_to(p2):.2f}")
print(f"ระยะ p1→p3 = {p1.distance_to(p3):.2f}")

# 3. HTTP Response
from typing import NamedTuple, Any

class HttpResponse(NamedTuple):
    status_code: int
    headers: dict
    body: Any
    
    @property
    def is_success(self):
        return 200 <= self.status_code < 300

def fake_api_call(endpoint):
    if endpoint == "/users":
        return HttpResponse(
            200,
            {"Content-Type": "application/json"},
            [{"id": 1, "name": "Alice"}]
        )
    return HttpResponse(404, {}, {"error": "Not found"})

response = fake_api_call("/users")
print(f"Status: {response.status_code}")
print(f"Success: {response.is_success}")
print(f"Body: {response.body}")

# 4. RGB Colors
Color = namedtuple("Color", ["r", "g", "b"])

RED = Color(255, 0, 0)
GREEN = Color(0, 255, 0)
BLUE = Color(0, 0, 255)

def blend_colors(c1, c2, ratio=0.5):
    """ผสมสี 2 สีเข้าด้วยกัน"""
    return Color(
        int(c1.r * ratio + c2.r * (1 - ratio)),
        int(c1.g * ratio + c2.g * (1 - ratio)),
        int(c1.b * ratio + c2.b * (1 - ratio)),
    )

purple = blend_colors(RED, BLUE)
print(f"Red + Blue = {purple}")  # Color(r=127, g=0, b=127)
```

---

## 8. ตัวอย่างโปรแกรมจริง: Geographic Data

```python
"""
ระบบจัดการข้อมูลทางภูมิศาสตร์โดยใช้ Named Tuples
"""
import math
from typing import NamedTuple, List

class Coordinate(NamedTuple):
    latitude: float   # ละติจูด
    longitude: float  # ลองจิจูด

class City(NamedTuple):
    name: str
    country: str
    coord: Coordinate
    population: int

EARTH_RADIUS_KM = 6371

def haversine_distance(c1: Coordinate, c2: Coordinate) -> float:
    """คำนวณระยะทางระหว่าง 2 จุดบนโลก (กิโลเมตร)"""
    lat1, lon1 = math.radians(c1.latitude), math.radians(c1.longitude)
    lat2, lon2 = math.radians(c2.latitude), math.radians(c2.longitude)
    
    dlat = lat2 - lat1
    dlon = lon2 - lon1
    
    a = math.sin(dlat/2)**2 + math.cos(lat1) * math.cos(lat2) * math.sin(dlon/2)**2
    c = 2 * math.asin(math.sqrt(a))
    
    return EARTH_RADIUS_KM * c

# ข้อมูลเมืองต่างๆ
cities: List[City] = [
    City("Bangkok", "Thailand", Coordinate(13.7563, 100.5018), 10_539_000),
    City("Singapore", "Singapore", Coordinate(1.3521, 103.8198), 5_850_000),
    City("Tokyo", "Japan", Coordinate(35.6762, 139.6503), 13_960_000),
    City("Sydney", "Australia", Coordinate(-33.8688, 151.2093), 5_312_000),
    City("Dubai", "UAE", Coordinate(25.2048, 55.2708), 3_478_000),
    City("London", "UK", Coordinate(51.5074, -0.1278), 8_982_000),
    City("New York", "USA", Coordinate(40.7128, -74.0060), 8_336_000),
]

# แสดงข้อมูล
print("=== เมืองหลักของโลก ===")
for city in sorted(cities, key=lambda c: c.population, reverse=True):
    print(f"  {city.name:15} ({city.country:12}) ประชากร: {city.population:>12,} คน")

# หาระยะทางจากกรุงเทพ
bangkok = cities[0]
print(f"\n=== ระยะทางจาก{bangkok.name} ===")
distances = []
for city in cities[1:]:
    dist = haversine_distance(bangkok.coord, city.coord)
    distances.append((city, dist))

for city, dist in sorted(distances, key=lambda x: x[1]):
    print(f"  → {city.name:15}: {dist:8.0f} กม.")

# หาเมืองที่ใกล้ที่สุดและไกลที่สุด
nearest = min(distances, key=lambda x: x[1])
farthest = max(distances, key=lambda x: x[1])
print(f"\nใกล้สุด: {nearest[0].name} ({nearest[1]:.0f} กม.)")
print(f"ไกลสุด: {farthest[0].name} ({farthest[1]:.0f} กม.)")
```

---

## 9. Exercises

### Exercise 1: Student Records

```python
"""
สร้างระบบเก็บข้อมูลนักเรียนด้วย NamedTuple:
1. Student มีฟิลด์: name, student_id, grades (tuple ของคะแนน)
2. ฟังก์ชัน calculate_gpa(student) - คำนวณ GPA
3. ฟังก์ชัน get_honor_roll(students) - คืน list นักเรียนที่ GPA >= 3.5
4. ฟังก์ชัน class_statistics(students) - สถิติชั้นเรียน
"""
from typing import NamedTuple, Tuple, List

class Student(NamedTuple):
    name: str
    student_id: str
    grades: Tuple[float, ...]  # ไม่จำกัดจำนวนคะแนน

def calculate_gpa(student: Student) -> float:
    """แปลงคะแนนเป็น GPA (4.0 scale)"""
    if not student.grades:
        return 0.0
    avg = sum(student.grades) / len(student.grades)
    # แปลง 0-100 เป็น 0-4.0
    if avg >= 90: return 4.0
    elif avg >= 80: return 3.0
    elif avg >= 70: return 2.0
    elif avg >= 60: return 1.0
    return 0.0

def get_honor_roll(students: List[Student]) -> List[Student]:
    return [s for s in students if calculate_gpa(s) >= 3.0]

def class_statistics(students: List[Student]) -> dict:
    gpas = [calculate_gpa(s) for s in students]
    return {
        "count": len(students),
        "average_gpa": sum(gpas) / len(gpas) if gpas else 0,
        "highest": max(gpas) if gpas else 0,
        "lowest": min(gpas) if gpas else 0,
        "honor_roll_count": sum(1 for g in gpas if g >= 3.0),
    }

# ทดสอบ
students = [
    Student("Alice", "S001", (92, 88, 95, 91, 89)),
    Student("Bob", "S002", (75, 72, 78, 70, 74)),
    Student("Charlie", "S003", (88, 85, 90, 87, 92)),
    Student("Diana", "S004", (55, 60, 58, 62, 57)),
    Student("Evan", "S005", (80, 82, 79, 85, 81)),
]

print("=== GPA นักเรียนทั้งหมด ===")
for student in students:
    gpa = calculate_gpa(student)
    print(f"  {student.name} ({student.student_id}): GPA {gpa:.1f}")

print("\n=== Honor Roll ===")
for student in get_honor_roll(students):
    gpa = calculate_gpa(student)
    print(f"  {student.name}: GPA {gpa:.1f}")

stats = class_statistics(students)
print(f"\n=== สถิติชั้น ===")
print(f"  จำนวน: {stats['count']} คน")
print(f"  เฉลี่ย GPA: {stats['average_gpa']:.2f}")
print(f"  Honor Roll: {stats['honor_roll_count']} คน")
```

### Exercise 2: Inventory System

```python
"""
ระบบคลังสินค้าด้วย NamedTuple:
1. Product(id, name, price, quantity, category)
2. หาสินค้าที่ใกล้หมด (quantity < 10)
3. คำนวณมูลค่าสินค้าทั้งหมด
4. จัดกลุ่มตาม category
"""
from typing import NamedTuple, List
from collections import defaultdict

class Product(NamedTuple):
    id: str
    name: str
    price: float
    quantity: int
    category: str

def low_stock(products: List[Product], threshold: int = 10) -> List[Product]:
    return [p for p in products if p.quantity < threshold]

def total_inventory_value(products: List[Product]) -> float:
    return sum(p.price * p.quantity for p in products)

def group_by_category(products: List[Product]) -> dict:
    groups = defaultdict(list)
    for p in products:
        groups[p.category].append(p)
    return dict(groups)

# ทดสอบ
inventory = [
    Product("P001", "Laptop", 35000, 15, "Electronics"),
    Product("P002", "Mouse", 599, 3, "Electronics"),
    Product("P003", "Keyboard", 1299, 8, "Electronics"),
    Product("P004", "Desk", 8500, 5, "Furniture"),
    Product("P005", "Chair", 4500, 12, "Furniture"),
    Product("P006", "Pen", 25, 200, "Stationery"),
    Product("P007", "Notebook", 89, 45, "Stationery"),
]

print("=== สินค้าที่ใกล้หมด ===")
for p in low_stock(inventory):
    print(f"  [{p.id}] {p.name}: เหลือ {p.quantity} ชิ้น")

print(f"\nมูลค่าคลังสินค้าทั้งหมด: {total_inventory_value(inventory):,.2f} บาท")

groups = group_by_category(inventory)
print("\n=== สินค้าแยกตาม Category ===")
for cat, products in sorted(groups.items()):
    cat_value = sum(p.price * p.quantity for p in products)
    print(f"  {cat}: {len(products)} รายการ (มูลค่า {cat_value:,.2f} บาท)")
```

### Exercise 3: Playing Cards

```python
"""
สร้างระบบไพ่โดยใช้ tuple:
1. สร้างสำรับไพ่ 52 ใบ
2. สุ่มแจกไพ่
3. เปรียบเทียบมือไพ่
"""
import random
from collections import namedtuple

Card = namedtuple("Card", ["rank", "suit"])

RANKS = ("2", "3", "4", "5", "6", "7", "8", "9", "10", "J", "Q", "K", "A")
SUITS = ("♠", "♥", "♦", "♣")
RANK_VALUES = {r: i for i, r in enumerate(RANKS, 2)}

def create_deck():
    """สร้างสำรับไพ่ 52 ใบ"""
    return tuple(Card(rank, suit) for suit in SUITS for rank in RANKS)

def shuffle_deck(deck):
    """สุ่มไพ่ (คืน list)"""
    deck_list = list(deck)
    random.shuffle(deck_list)
    return tuple(deck_list)

def deal_hand(deck, n=5):
    """แจกไพ่ n ใบ คืน (hand, remaining_deck)"""
    return deck[:n], deck[n:]

def card_value(card):
    """ค่าของไพ่"""
    return RANK_VALUES[card.rank]

def hand_score(hand):
    """คะแนนของมือไพ่ (ผลรวมค่าไพ่)"""
    return sum(card_value(c) for c in hand)

def format_card(card):
    return f"{card.rank}{card.suit}"

# ทดสอบ
deck = create_deck()
print(f"สำรับไพ่: {len(deck)} ใบ")

deck = shuffle_deck(deck)
hand1, deck = deal_hand(deck)
hand2, deck = deal_hand(deck)

print(f"\nมือที่ 1: {' '.join(format_card(c) for c in hand1)} (คะแนน: {hand_score(hand1)})")
print(f"มือที่ 2: {' '.join(format_card(c) for c in hand2)} (คะแนน: {hand_score(hand2)})")

if hand_score(hand1) > hand_score(hand2):
    print("มือที่ 1 ชนะ!")
elif hand_score(hand2) > hand_score(hand1):
    print("มือที่ 2 ชนะ!")
else:
    print("เสมอกัน!")

print(f"\nไพ่ที่เหลือในสำรับ: {len(deck)} ใบ")
```

---

## 10. สรุป Part 008

### สิ่งที่เรียนรู้:

✅ **Tuple** - sequence ที่ไม่เปลี่ยนแปลง (immutable)  
✅ **สร้าง Tuple** - `()`, `tuple()`, tuple packing  
✅ **Immutability** - ป้องกันการแก้ไขข้อมูลโดยไม่ตั้งใจ  
✅ **Unpacking** - `x, y = point`, extended unpacking ด้วย `*`  
✅ **Named Tuples** - `namedtuple()` และ `typing.NamedTuple`  
✅ **Return values** - คืนหลายค่าจากฟังก์ชัน  
✅ **Hashable** - ใช้เป็น dict key และ set member ได้  
✅ **Memory** - ใช้ memory น้อยกว่า list  

### เมื่อไหรใช้ Tuple vs List:

| ใช้ Tuple | ใช้ List |
|-----------|----------|
| ข้อมูลไม่เปลี่ยน | ข้อมูลเปลี่ยนได้ |
| Heterogeneous data | Homogeneous data |
| Dict key | - |
| Return multiple values | รวบรวม results |
| Performance สำคัญ | ต้องการ flexibility |

### Quick Reference:

```python
# สร้าง
t = (1, 2, 3)
t = 1, 2, 3       # packing
t = (42,)         # single element

# เข้าถึง
t[0], t[-1]
t[1:3]

# Unpacking
a, b, c = t
first, *rest = t

# Named Tuple
from typing import NamedTuple
class Point(NamedTuple):
    x: float
    y: float

p = Point(3.0, 4.0)
p.x, p.y
p._replace(x=10.0)
p._asdict()
```

---

## ➡️ ถัดไป: Part 009 - Dictionaries

ใน Part ถัดไป เราจะเรียนรู้:
- Dictionary methods ครบทุก method
- Nested dictionaries
- defaultdict, Counter, OrderedDict
- Dictionary comprehension ขั้นสูง
- Merge dicts ด้วย `|=` operator (Python 3.9+)

---

*Part 008/100+ | Python Course - Beginner to World-Class*
