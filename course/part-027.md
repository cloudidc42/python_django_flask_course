# Part 027: Type Hints และ mypy
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Type Hints และประโยชน์ของการใช้
- ใช้ annotations พื้นฐาน: int, str, float, bool, None
- ใช้ Optional, Union, List, Dict, Tuple, Set
- เข้าใจ TypeVar และ Generic types
- สร้าง Protocol สำหรับ structural subtyping
- ใช้ dataclasses และ TypedDict
- รัน mypy เพื่อ type checking แบบ static

---

## 1. ทำไมต้องใช้ Type Hints?

Type Hints (PEP 484) เพิ่มข้อมูล type ให้กับโค้ด Python โดยไม่เปลี่ยนพฤติกรรมการทำงาน

```python
# ❌ ไม่มี type hints - ไม่รู้ว่าแต่ละ argument เป็น type อะไร
def add(a, b):
    return a + b

def greet(name):
    return "Hello, " + name

# ✅ มี type hints - ชัดเจนว่าใส่อะไรและได้อะไรกลับมา
def add(a: int, b: int) -> int:
    return a + b

def greet(name: str) -> str:
    return "Hello, " + name

# ข้อดีของ Type Hints:
# 1. IDE autocomplete ดีขึ้น
# 2. ตรวจจับ bugs ได้เร็วขึ้น (ก่อน runtime)
# 3. Code documentation ในตัว
# 4. ใช้กับ mypy สำหรับ static type checking
# 5. ทีมทำงานร่วมกันได้ง่ายขึ้น

# Type hints ไม่ได้ enforce ที่ runtime (Python ยังคง dynamic)
result = add("hello", "world")  # ไม่ error ที่ runtime แต่ mypy จะแจ้งเตือน
```

---

## 2. Basic Type Annotations

```python
# Variable annotations
name: str = "Alice"
age: int = 30
height: float = 5.6
is_active: bool = True
data: bytes = b"hello"

# Function annotations
def divide(a: float, b: float) -> float:
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b

def say_hello(name: str) -> None:
    """None หมายความว่า function ไม่ return ค่า"""
    print(f"Hello, {name}!")

# หลายๆ parameters
def create_user(
    username: str,
    email: str,
    age: int,
    is_admin: bool = False
) -> dict:
    return {
        "username": username,
        "email": email,
        "age": age,
        "is_admin": is_admin
    }

# การใช้งาน
user = create_user("alice", "alice@example.com", 25)
print(user)
# {'username': 'alice', 'email': 'alice@example.com', 'age': 25, 'is_admin': False}

# Type alias - ตั้งชื่อ type ที่ใช้บ่อย
UserId = int
Username = str

def get_user(user_id: UserId) -> Username:
    users = {1: "alice", 2: "bob"}
    return users.get(user_id, "unknown")

print(get_user(1))  # alice
```

---

## 3. Optional และ Union

```python
from typing import Optional, Union

# Optional[X] เทียบเท่ากับ Union[X, None]
# ใช้เมื่อ value อาจเป็น None

def find_user(user_id: int) -> Optional[str]:
    """คืนค่า username หรือ None ถ้าไม่เจอ"""
    users = {1: "alice", 2: "bob", 3: "charlie"}
    return users.get(user_id)  # คืน None ถ้าไม่มี key

result = find_user(1)
if result is not None:
    print(f"Found: {result.upper()}")  # mypy รู้ว่า result เป็น str ณ จุดนี้

result2 = find_user(99)
print(result2)  # None

# Union[X, Y] - รับได้หลาย types
def process_input(data: Union[str, int, float]) -> str:
    """รับ string, int, หรือ float แล้วแปลงเป็น string"""
    return str(data)

print(process_input("hello"))  # hello
print(process_input(42))       # 42
print(process_input(3.14))     # 3.14

# Python 3.10+ ใช้ | แทน Union
# def process(data: str | int | float) -> str: ...

# Optional กับ default value
def connect(
    host: str,
    port: int = 8080,
    timeout: Optional[float] = None
) -> bool:
    """เชื่อมต่อ server โดย timeout=None หมายถึงรอนานเท่าไหร่ก็ได้"""
    print(f"Connecting to {host}:{port}")
    if timeout:
        print(f"Timeout: {timeout}s")
    return True

connect("localhost")              # ใช้ default port
connect("example.com", 443)      # กำหนด port
connect("api.test.com", timeout=30.0)  # กำหนด timeout

# Nested Optional
from typing import Optional, List

def get_first_name(user: Optional[dict]) -> Optional[str]:
    """ดึงชื่อจาก user dict ที่อาจเป็น None"""
    if user is None:
        return None
    return user.get("first_name")
```

---

## 4. Collection Types: List, Dict, Tuple, Set

```python
from typing import List, Dict, Tuple, Set, Sequence, Mapping

# List[X] - list ที่มี elements เป็น type X
def sum_numbers(numbers: List[int]) -> int:
    return sum(numbers)

def get_names(users: List[Dict[str, str]]) -> List[str]:
    return [u["name"] for u in users]

# Dict[K, V] - dictionary ที่ key เป็น K และ value เป็น V
def count_words(text: str) -> Dict[str, int]:
    word_count: Dict[str, int] = {}
    for word in text.split():
        word_count[word] = word_count.get(word, 0) + 1
    return word_count

result = count_words("the cat sat on the mat the cat")
print(result)  # {'the': 3, 'cat': 2, 'sat': 1, 'on': 1, 'mat': 1}

# Tuple - fixed-size, immutable sequences
# Tuple[X, Y, Z] - tuple ที่มี 3 elements: X, Y, Z
def get_coordinates() -> Tuple[float, float]:
    """คืนค่า (latitude, longitude)"""
    return (13.7563, 100.5018)  # Bangkok

lat, lon = get_coordinates()
print(f"Lat: {lat}, Lon: {lon}")

# Tuple[X, ...] - tuple ที่มี elements เป็น X จำนวนเท่าใดก็ได้
def sum_tuple(values: Tuple[int, ...]) -> int:
    return sum(values)

print(sum_tuple((1, 2, 3, 4, 5)))  # 15

# Set[X]
def unique_tags(posts: List[Dict]) -> Set[str]:
    tags: Set[str] = set()
    for post in posts:
        tags.update(post.get("tags", []))
    return tags

posts = [
    {"title": "A", "tags": ["python", "coding"]},
    {"title": "B", "tags": ["python", "web"]},
]
print(unique_tags(posts))  # {'python', 'coding', 'web'}

# Sequence - อ่านได้อย่างเดียว (list, tuple, str, etc.)
def first_item(seq: Sequence[int]) -> Optional[int]:
    return seq[0] if seq else None

print(first_item([1, 2, 3]))   # 1
print(first_item((4, 5, 6)))   # 4
print(first_item([]))           # None

# Python 3.9+ ใช้ built-in types โดยตรง
def modern_types(
    numbers: list[int],
    mapping: dict[str, int],
    point: tuple[float, float]
) -> set[str]:
    return {str(n) for n in numbers}
```

---

## 5. Callable, Iterator, Generator

```python
from typing import Callable, Iterator, Generator, Any

# Callable[[ArgTypes...], ReturnType]
def apply_twice(func: Callable[[int], int], value: int) -> int:
    """เรียก function สองครั้งกับ value"""
    return func(func(value))

def double(x: int) -> int:
    return x * 2

print(apply_twice(double, 3))  # 12 (3 → 6 → 12)

# Callable ที่รับ args หลายตัว
def transform(
    data: list,
    transform_fn: Callable[[Any], Any]
) -> list:
    return [transform_fn(item) for item in data]

result = transform([1, 2, 3], lambda x: x ** 2)
print(result)  # [1, 4, 9]

# Iterator[X]
def count_up(start: int, end: int) -> Iterator[int]:
    """Generator ที่ iterate ตั้งแต่ start ถึง end"""
    current = start
    while current <= end:
        yield current
        current += 1

for n in count_up(1, 5):
    print(n, end=" ")  # 1 2 3 4 5
print()

# Generator[YieldType, SendType, ReturnType]
def fibonacci() -> Generator[int, None, None]:
    """Infinite fibonacci generator"""
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

fib = fibonacci()
first_10 = [next(fib) for _ in range(10)]
print(first_10)  # [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

# Higher-order functions
from typing import Callable, TypeVar

T = TypeVar('T')
S = TypeVar('S')

def map_func(items: List[T], fn: Callable[[T], S]) -> List[S]:
    """Generic map function"""
    return [fn(item) for item in items]

numbers = [1, 2, 3, 4, 5]
strings = map_func(numbers, str)
print(strings)  # ['1', '2', '3', '4', '5']

doubled = map_func(numbers, lambda x: x * 2)
print(doubled)  # [2, 4, 6, 8, 10]
```

---

## 6. TypeVar และ Generic Types

```python
from typing import TypeVar, Generic, List, Optional

# TypeVar - สร้าง type variable สำหรับ generic functions
T = TypeVar('T')

def first(items: List[T]) -> Optional[T]:
    """คืนค่าตัวแรกของ list หรือ None ถ้า empty"""
    return items[0] if items else None

print(first([1, 2, 3]))         # 1 (inferred as int)
print(first(["a", "b", "c"]))   # 'a' (inferred as str)
print(first([]))                 # None

def identity(value: T) -> T:
    """คืนค่าเดิมที่รับมา"""
    return value

x: int = identity(42)
s: str = identity("hello")

# TypeVar กับ constraint
Numeric = TypeVar('Numeric', int, float)

def add_numbers(a: Numeric, b: Numeric) -> Numeric:
    """รับได้เฉพาะ int หรือ float"""
    return a + b

print(add_numbers(1, 2))        # 3
print(add_numbers(1.5, 2.5))    # 4.0
# add_numbers("a", "b")  # mypy จะ error

# Generic Class
class Stack(Generic[T]):
    """Stack data structure แบบ generic"""
    
    def __init__(self) -> None:
        self._items: List[T] = []
    
    def push(self, item: T) -> None:
        self._items.append(item)
    
    def pop(self) -> Optional[T]:
        if not self._items:
            return None
        return self._items.pop()
    
    def peek(self) -> Optional[T]:
        if not self._items:
            return None
        return self._items[-1]
    
    def is_empty(self) -> bool:
        return len(self._items) == 0
    
    def size(self) -> int:
        return len(self._items)

# Stack[int] - stack ของ integers
int_stack: Stack[int] = Stack()
int_stack.push(1)
int_stack.push(2)
int_stack.push(3)
print(int_stack.pop())   # 3
print(int_stack.peek())  # 2

# Stack[str] - stack ของ strings
str_stack: Stack[str] = Stack()
str_stack.push("hello")
str_stack.push("world")
print(str_stack.pop())   # world

# Generic class ที่มีหลาย type parameters
class Pair(Generic[T, S]):
    """คู่ของ values ต่าง type"""
    
    def __init__(self, first: T, second: S) -> None:
        self.first = first
        self.second = second
    
    def swap(self) -> 'Pair[S, T]':
        return Pair(self.second, self.first)
    
    def __repr__(self) -> str:
        return f"Pair({self.first!r}, {self.second!r})"

pair = Pair(1, "one")
print(pair)         # Pair(1, 'one')
swapped = pair.swap()
print(swapped)      # Pair('one', 1)
```

---

## 7. Protocol: Structural Subtyping

```python
from typing import Protocol, runtime_checkable

# Protocol กำหนด "interface" โดยดูที่ methods ที่มี
# ไม่ต้อง inherit จาก Protocol class

class Drawable(Protocol):
    """Protocol สำหรับ objects ที่ draw ได้"""
    def draw(self) -> str: ...

class Resizable(Protocol):
    """Protocol สำหรับ objects ที่ resize ได้"""
    def resize(self, factor: float) -> None: ...

# Classes ที่ implement protocols โดยไม่ต้อง inherit
class Circle:
    def __init__(self, radius: float) -> None:
        self.radius = radius
    
    def draw(self) -> str:
        return f"⭕ Circle(radius={self.radius})"
    
    def resize(self, factor: float) -> None:
        self.radius *= factor

class Rectangle:
    def __init__(self, width: float, height: float) -> None:
        self.width = width
        self.height = height
    
    def draw(self) -> str:
        return f"▭ Rectangle({self.width}x{self.height})"
    
    def resize(self, factor: float) -> None:
        self.width *= factor
        self.height *= factor

class Text:
    def __init__(self, content: str) -> None:
        self.content = content
    
    def draw(self) -> str:
        return f"📝 Text: {self.content}"

# Function ที่รับ Drawable protocol
def render(shape: Drawable) -> None:
    """render ได้ทุก object ที่มี draw() method"""
    print(shape.draw())

# ทุก class สามารถส่งได้เพราะมี draw() method
render(Circle(5.0))            # ⭕ Circle(radius=5.0)
render(Rectangle(3.0, 4.0))   # ▭ Rectangle(3.0x4.0)
render(Text("Hello"))          # 📝 Text: Hello

# render ด้วย Resizable protocol
def scale_all(shapes: List[Resizable], factor: float) -> None:
    for shape in shapes:
        shape.resize(factor)

shapes = [Circle(5.0), Rectangle(3.0, 4.0)]
scale_all(shapes, 2.0)
for s in shapes:
    print(s.draw())
# ⭕ Circle(radius=10.0)
# ▭ Rectangle(6.0x8.0)

# @runtime_checkable ทำให้ใช้ isinstance() ได้
@runtime_checkable
class Closeable(Protocol):
    def close(self) -> None: ...

class FileWrapper:
    def close(self) -> None:
        print("File closed")

fw = FileWrapper()
print(isinstance(fw, Closeable))  # True

# Protocol ที่มี attributes ด้วย
class HasName(Protocol):
    name: str
    
    def greet(self) -> str: ...

class Person:
    def __init__(self, name: str) -> None:
        self.name = name
    
    def greet(self) -> str:
        return f"Hi, I'm {self.name}"

def introduce(entity: HasName) -> None:
    print(entity.greet())

introduce(Person("Alice"))  # Hi, I'm Alice
```

---

## 8. dataclasses และ TypedDict

```python
from dataclasses import dataclass, field
from typing import TypedDict, List, Optional

# TypedDict - dict ที่มี type ที่กำหนดไว้
class UserDict(TypedDict):
    id: int
    name: str
    email: str

class UserOptionalDict(TypedDict, total=False):
    """total=False ทำให้ทุก key เป็น optional"""
    id: int
    name: str
    email: str
    phone: str

# ใช้ TypedDict
user: UserDict = {
    "id": 1,
    "name": "Alice",
    "email": "alice@example.com"
}

# mypy จะ error ถ้า field ไม่ถูก
# bad_user: UserDict = {"id": 1}  # Error: missing 'name', 'email'

def get_user_name(user: UserDict) -> str:
    return user["name"]

print(get_user_name(user))  # Alice

# Combining TypedDict
class BaseUser(TypedDict):
    id: int
    name: str

class AdminUser(BaseUser):
    role: str
    permissions: List[str]

admin: AdminUser = {
    "id": 1,
    "name": "Admin",
    "role": "superadmin",
    "permissions": ["read", "write", "delete"]
}

# dataclass พื้นฐาน
@dataclass
class Point:
    x: float
    y: float
    
    def distance_to(self, other: 'Point') -> float:
        return ((self.x - other.x) ** 2 + (self.y - other.y) ** 2) ** 0.5

p1 = Point(0.0, 0.0)
p2 = Point(3.0, 4.0)
print(p1.distance_to(p2))  # 5.0
print(p1)  # Point(x=0.0, y=0.0)

# dataclass กับ default values
@dataclass
class Config:
    host: str = "localhost"
    port: int = 8080
    debug: bool = False
    tags: List[str] = field(default_factory=list)  # mutable default ต้องใช้ field()

cfg = Config()
print(cfg)  # Config(host='localhost', port=8080, debug=False, tags=[])

cfg2 = Config(host="0.0.0.0", port=443, tags=["prod"])
print(cfg2)  # Config(host='0.0.0.0', port=443, debug=False, tags=['prod'])
```

---

## 9. ตัวอย่างจริง: Type-Safe API Client

```python
from typing import TypedDict, Optional, List, Union, Generic, TypeVar
from dataclasses import dataclass, field
import json

# Type definitions
class ApiError(TypedDict):
    code: int
    message: str
    details: Optional[str]

class PaginationMeta(TypedDict):
    total: int
    page: int
    per_page: int
    total_pages: int

T = TypeVar('T')

@dataclass
class ApiResponse(Generic[T]):
    """Generic API response wrapper"""
    success: bool
    data: Optional[T]
    error: Optional[ApiError] = None
    meta: Optional[PaginationMeta] = None
    
    def is_ok(self) -> bool:
        return self.success and self.data is not None
    
    def unwrap(self) -> T:
        """ดึง data หรือ raise error ถ้าไม่ success"""
        if not self.success or self.data is None:
            err_msg = self.error["message"] if self.error else "Unknown error"
            raise ValueError(f"API Error: {err_msg}")
        return self.data

class UserProfile(TypedDict):
    id: int
    username: str
    email: str
    full_name: str
    is_active: bool

class Post(TypedDict):
    id: int
    title: str
    content: str
    author_id: int
    tags: List[str]

# Simulated API functions
def fetch_user(user_id: int) -> ApiResponse[UserProfile]:
    """ดึงข้อมูล user จาก API"""
    # Simulate API call
    if user_id == 1:
        user: UserProfile = {
            "id": 1,
            "username": "alice",
            "email": "alice@example.com",
            "full_name": "Alice Smith",
            "is_active": True
        }
        return ApiResponse(success=True, data=user)
    else:
        error: ApiError = {
            "code": 404,
            "message": "User not found",
            "details": f"No user with id={user_id}"
        }
        return ApiResponse(success=False, data=None, error=error)

def fetch_posts(
    page: int = 1,
    per_page: int = 10
) -> ApiResponse[List[Post]]:
    """ดึง list ของ posts"""
    posts: List[Post] = [
        {"id": 1, "title": "Python Tips", "content": "...", "author_id": 1, "tags": ["python"]},
        {"id": 2, "title": "Type Hints", "content": "...", "author_id": 1, "tags": ["python", "types"]},
    ]
    meta: PaginationMeta = {
        "total": 2,
        "page": page,
        "per_page": per_page,
        "total_pages": 1
    }
    return ApiResponse(success=True, data=posts, meta=meta)

# ใช้งาน
response = fetch_user(1)
if response.is_ok():
    user = response.unwrap()
    print(f"User: {user['full_name']} ({user['email']})")
    # User: Alice Smith (alice@example.com)

response2 = fetch_user(99)
if not response2.is_ok():
    print(f"Error: {response2.error['message']}")  # Error: User not found

posts_response = fetch_posts(page=1, per_page=10)
if posts_response.is_ok():
    posts = posts_response.unwrap()
    for post in posts:
        print(f"- {post['title']} [{', '.join(post['tags'])}]")
    # - Python Tips [python]
    # - Type Hints [python, types]
    
    if posts_response.meta:
        print(f"Page {posts_response.meta['page']}/{posts_response.meta['total_pages']}")
        # Page 1/1
```

---

## 10. รัน mypy

```python
# install: pip install mypy

# สร้างไฟล์ example.py ที่มี type errors
"""
# example.py
def add(a: int, b: int) -> int:
    return a + b

# Error 1: argument type mismatch
result = add("hello", "world")

# Error 2: wrong return type usage
x: int = add(1, 2)
y: str = add(1, 2)  # int cannot be assigned to str

# Error 3: Optional not handled
from typing import Optional

def find(items: list, target: int) -> Optional[int]:
    for i, item in enumerate(items):
        if item == target:
            return i
    return None

index = find([1, 2, 3], 2)
print(index + 1)  # Error: Optional[int] not supported for +
"""

# รัน mypy:
# $ mypy example.py
# example.py:5: error: Argument 1 to "add" has incompatible type "str"; expected "int"
# example.py:5: error: Argument 2 to "add" has incompatible type "str"; expected "int"
# example.py:9: error: Incompatible types in assignment (expression has type "int", variable has type "str")
# example.py:22: error: Unsupported left operand type for + ("Optional[int]")
```

```python
# mypy configuration ใน mypy.ini หรือ pyproject.toml

# mypy.ini:
"""
[mypy]
python_version = 3.11
warn_return_any = True
warn_unused_configs = True
disallow_untyped_defs = True
check_untyped_defs = True
"""

# pyproject.toml:
"""
[tool.mypy]
python_version = "3.11"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
"""

# คำสั่ง mypy ที่มีประโยชน์:
# mypy myfile.py                    # check ไฟล์เดียว
# mypy mypackage/                   # check ทั้ง package
# mypy --strict myfile.py           # strict mode (เข้มงวดมาก)
# mypy --ignore-missing-imports f   # ignore เมื่อ library ไม่มี stubs
# mypy --show-error-codes file.py   # แสดง error codes

# ตัวอย่าง code ที่ผ่าน mypy
from typing import Optional, List, Dict

def process_data(
    items: List[int],
    multiplier: float = 1.0,
    label: Optional[str] = None
) -> Dict[str, float]:
    results: Dict[str, float] = {}
    for i, item in enumerate(items):
        key = f"{label}_{i}" if label else str(i)
        results[key] = item * multiplier
    return results

data = process_data([1, 2, 3], multiplier=2.5, label="item")
print(data)  # {'item_0': 2.5, 'item_1': 5.0, 'item_2': 7.5}
```

---

## 11. Advanced Type Features

```python
from typing import Final, ClassVar, Literal, Annotated, overload

# Final - ค่าที่ไม่สามารถ reassign ได้
MAX_SIZE: Final = 100
DATABASE_URL: Final[str] = "postgresql://localhost/mydb"

# MAX_SIZE = 200  # mypy error: Cannot assign to final name

# Literal - รับได้เฉพาะค่าที่กำหนด
Direction = Literal["north", "south", "east", "west"]
HttpMethod = Literal["GET", "POST", "PUT", "DELETE", "PATCH"]

def move(direction: Direction, steps: int) -> str:
    return f"Moving {direction} {steps} steps"

def make_request(method: HttpMethod, url: str) -> dict:
    return {"method": method, "url": url}

print(move("north", 5))           # Moving north 5 steps
print(make_request("GET", "/api"))  # {'method': 'GET', 'url': '/api'}
# move("up", 3)  # mypy error: invalid direction

# ClassVar - class variable (ไม่ใช่ instance variable)
class Counter:
    count: ClassVar[int] = 0  # shared across all instances
    
    def __init__(self, name: str) -> None:
        Counter.count += 1
        self.name = name
        self.id: int = Counter.count  # instance variable

c1 = Counter("A")
c2 = Counter("B")
print(Counter.count)  # 2
print(c1.id, c2.id)   # 1 2

# overload - function ที่มีหลาย signatures
from typing import overload, Union

@overload
def double(x: int) -> int: ...
@overload
def double(x: str) -> str: ...
@overload
def double(x: float) -> float: ...

def double(x: Union[int, str, float]) -> Union[int, str, float]:
    if isinstance(x, str):
        return x * 2
    return x * 2

result_int: int = double(5)        # 10
result_str: str = double("hi")     # "hihi"
result_float: float = double(2.5)  # 5.0

print(result_int, result_str, result_float)  # 10 hihi 5.0

# Annotated - เพิ่ม metadata ให้กับ types
from typing import Annotated

# ใช้สำหรับ runtime validation libraries (เช่น Pydantic)
PositiveInt = Annotated[int, "must be > 0"]
EmailStr = Annotated[str, "must be valid email"]

def create_user_v2(
    age: Annotated[int, "must be between 18-120"],
    email: Annotated[str, "must contain @"]
) -> dict:
    return {"age": age, "email": email}
```

---

## 12. สรุป Part 027

✅ **Type Hints ช่วยให้โค้ดอ่านง่ายขึ้น** และ IDE ให้ autocomplete ที่ดีขึ้น  
✅ **Basic types**: int, str, float, bool, None  
✅ **Optional[X]** = ค่าอาจเป็น None  
✅ **Union[X, Y]** = รับได้หลาย types  
✅ **Collection types**: List, Dict, Tuple, Set  
✅ **TypeVar + Generic** = สร้าง generic functions และ classes  
✅ **Protocol** = structural subtyping ไม่ต้อง inherit  
✅ **TypedDict** = dict ที่มี type ที่กำหนด  
✅ **mypy** = static type checker สำหรับ Python  
✅ **Final, Literal, ClassVar** = advanced type features  

---

## ➡️ ถัดไป: Part 028 - Dataclasses

*Part 027/100+ | Python Course - Beginner to World-Class*
