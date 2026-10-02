# Part 014: Modules และ Packages
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ import statements ทุกรูปแบบได้
- สร้าง Packages ของตัวเองได้
- เข้าใจ sys.path และ __name__ ได้
- ใช้งาน Standard Library Modules ที่สำคัญได้
- จัดโครงสร้าง Project ได้อย่างถูกต้อง

---

## 1. Module คืออะไร?

```python
# Module = ไฟล์ Python (.py) ที่มี functions, classes, variables
# Package = directory ที่มี __init__.py

# โครงสร้างโปรเจกต์ตัวอย่าง:
# my_project/
# ├── main.py
# ├── utils.py          ← module
# ├── models/           ← package
# │   ├── __init__.py
# │   ├── user.py
# │   └── product.py
# └── services/         ← package
#     ├── __init__.py
#     └── email_service.py

# สร้าง module ง่ายๆ
# utils.py
"""
Utility functions สำหรับโปรเจกต์
"""

# Module level variables
VERSION = "1.0.0"
AUTHOR = "Alice"

def add(a, b):
    """บวกเลข 2 ตัว"""
    return a + b

def multiply(a, b):
    """คูณเลข 2 ตัว"""
    return a * b

class Calculator:
    def __init__(self, name="default"):
        self.name = name
        self.history = []
    
    def calculate(self, op, a, b):
        if op == "+": result = a + b
        elif op == "-": result = a - b
        elif op == "*": result = a * b
        elif op == "/":
            if b == 0:
                raise ZeroDivisionError("หารด้วย 0 ไม่ได้")
            result = a / b
        else:
            raise ValueError(f"Operation ไม่รู้จัก: {op}")
        
        self.history.append(f"{a} {op} {b} = {result}")
        return result

if __name__ == "__main__":
    # รันเฉพาะเมื่อเรียกไฟล์นี้โดยตรง
    print("Testing utils.py")
    print(add(3, 4))
    print(multiply(5, 6))
```

---

## 2. Import Statements

```python
# รูปแบบต่างๆ ของ import

# 1. import module
import os
import sys
import math

# ใช้ด้วย module.name
print(os.getcwd())
print(math.pi)
print(sys.version)

# 2. import with alias
import numpy as np          # convention
import pandas as pd         # convention
import datetime as dt

# ใช้ด้วย alias.name
today = dt.date.today()
print(today)

# 3. from module import name
from math import pi, sqrt, ceil, floor
from os.path import join, exists, dirname

print(pi)          # ไม่ต้องใส่ math.
print(sqrt(16))    # 4.0
print(ceil(4.3))   # 5

# 4. from module import name as alias
from datetime import datetime as DateTime
from collections import OrderedDict as OD

now = DateTime.now()
print(now)

# 5. from module import * (ไม่แนะนำ)
# from math import *  # นำ names ทั้งหมดมา - อาจชนกับ names เดิม

# 6. import ในฟังก์ชัน (lazy import)
def process_image(filepath):
    # import เฉพาะเมื่อใช้งาน (ลด startup time)
    from PIL import Image  # สมมุติมี Pillow
    return None  # placeholder

# ตัวอย่างไฟล์ main.py
# จำลองการ import utils
print("\n--- import simulation ---")

# สร้างไฟล์ utils.py ชั่วคราวเพื่อทดสอบ
import tempfile
import sys
from pathlib import Path

# สร้าง utils module ชั่วคราว
utils_code = '''
VERSION = "1.0.0"

def greet(name):
    return f"สวัสดี, {name}!"

def calculate(a, op, b):
    ops = {"+": a+b, "-": a-b, "*": a*b, "/": a/b if b != 0 else None}
    return ops.get(op, None)
'''

tmp_dir = Path(tempfile.mkdtemp())
utils_path = tmp_dir / "my_utils.py"
utils_path.write_text(utils_code)

# เพิ่ม directory เข้า sys.path
sys.path.insert(0, str(tmp_dir))

# ตอนนี้ import ได้
import my_utils

print(my_utils.VERSION)
print(my_utils.greet("Alice"))
print(my_utils.calculate(10, "+", 5))

# ทำ cleanup
sys.path.remove(str(tmp_dir))
import shutil
shutil.rmtree(tmp_dir)
```

---

## 3. sys.path และการค้นหา Module

```python
import sys

# sys.path คือ list ของ directories ที่ Python ค้นหา modules
print("=== sys.path ===")
for path in sys.path:
    print(f"  {path}")

# ลำดับการค้นหา:
# 1. Built-in modules (math, os, sys, ...)
# 2. Frozen modules
# 3. Current directory (หรือ directory ของ script)
# 4. PYTHONPATH environment variable
# 5. Standard library directories
# 6. Site-packages (pip installed)

# เพิ่ม path ชั่วคราว
sys.path.insert(0, "/path/to/my/modules")

# ดู built-in modules
print("\nBuilt-in modules:")
for mod in sorted(sys.builtin_module_names)[:10]:
    print(f"  {mod}")

# importlib - import ด้วย string name
import importlib

math_module = importlib.import_module("math")
print(f"\nmath.pi = {math_module.pi}")

# dynamic import
module_name = "collections"
mod = importlib.import_module(module_name)
Counter = getattr(mod, "Counter")
print(Counter("hello"))  # Counter({'l': 2, 'h': 1, 'e': 1, 'o': 1})

# reload module
importlib.reload(math_module)
```

---

## 4. \_\_name\_\_ == '\_\_main\_\_'

```python
# ทุก module มี __name__ attribute
# ถ้าเรียกตรงๆ: __name__ == '__main__'
# ถ้า import: __name__ == 'ชื่อ module'

print(f"__name__ = {__name__}")
# ถ้าเรียกตรง: __name__ = __main__
# ถ้า import: __name__ = ชื่อไฟล์ (ไม่มี .py)

# Pattern ที่ใช้บ่อย
def main():
    """ฟังก์ชัน main ของ program"""
    print("Program เริ่มทำงาน")
    result = run_application()
    print(f"ผลลัพธ์: {result}")

def run_application():
    return "SUCCESS"

# Functions และ Classes
def helper_function():
    return "helper"

class MyClass:
    pass

if __name__ == "__main__":
    # รันเฉพาะเมื่อเรียกตรงๆ ไม่รันเมื่อ import
    main()
    print("Running tests...")
    assert helper_function() == "helper"
    print("Tests passed!")

# ตัวอย่างการทดสอบ
print(f"\nCurrent __name__: {__name__}")
if __name__ == "__main__":
    print("This is the main script")
```

---

## 5. สร้าง Package

```python
# Package = directory + __init__.py

# โครงสร้าง:
# mypackage/
# ├── __init__.py          ← ทำให้เป็น package
# ├── core.py
# ├── utils.py
# └── models/
#     ├── __init__.py
#     ├── user.py
#     └── product.py

# จำลองการสร้าง package
import tempfile
import sys
from pathlib import Path

pkg_dir = Path(tempfile.mkdtemp())
mypackage = pkg_dir / "mypackage"
mypackage.mkdir()
(mypackage / "models").mkdir()

# สร้างไฟล์ต่างๆ
(mypackage / "__init__.py").write_text('''
"""
mypackage - ตัวอย่าง package

ใน __init__.py:
- กำหนด __version__, __author__
- import สิ่งที่ต้องการ expose
- ตั้งค่าต่างๆ
"""
__version__ = "1.0.0"
__author__ = "Alice"

# Import สิ่งที่ต้องการให้ user เห็น
from .core import Calculator
from .utils import format_number

# ซ่อน implementation details
__all__ = ["Calculator", "format_number"]
''')

(mypackage / "core.py").write_text('''
class Calculator:
    """เครื่องคิดเลข"""
    def add(self, a, b): return a + b
    def subtract(self, a, b): return a - b
    def multiply(self, a, b): return a * b
    def divide(self, a, b):
        if b == 0: raise ZeroDivisionError
        return a / b
''')

(mypackage / "utils.py").write_text('''
def format_number(n, decimals=2):
    """จัดรูปแบบตัวเลข"""
    return f"{n:,.{decimals}f}"

def percentage(part, total):
    """คำนวณเปอร์เซ็นต์"""
    return part / total * 100 if total else 0
''')

(mypackage / "models" / "__init__.py").write_text('''
from .user import User
from .product import Product
__all__ = ["User", "Product"]
''')

(mypackage / "models" / "user.py").write_text('''
class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email
    def __repr__(self):
        return f"User(name={self.name!r}, email={self.email!r})"
''')

(mypackage / "models" / "product.py").write_text('''
class Product:
    def __init__(self, name, price):
        self.name = name
        self.price = price
    def __repr__(self):
        return f"Product(name={self.name!r}, price={self.price})"
''')

# เพิ่ม pkg_dir เข้า sys.path
sys.path.insert(0, str(pkg_dir))

# ตอนนี้ import ได้
import mypackage
from mypackage import Calculator, format_number
from mypackage.models import User, Product
from mypackage.utils import percentage

print(f"mypackage version: {mypackage.__version__}")
print(f"author: {mypackage.__author__}")

calc = Calculator()
print(f"3 + 4 = {calc.add(3, 4)}")
print(f"100 / 4 = {calc.divide(100, 4)}")

print(format_number(1234567.89))
print(f"{percentage(45, 200):.1f}%")

user = User("Alice", "alice@example.com")
product = Product("Laptop", 35000)
print(user)
print(product)

# Import จาก subpackage
from mypackage.models.user import User as UserModel
u = UserModel("Bob", "bob@example.com")
print(u)

# Cleanup
sys.path.remove(str(pkg_dir))
import shutil
shutil.rmtree(pkg_dir)
```

---

## 6. Relative vs Absolute Imports

```python
# Absolute import (แนะนำ) - path เต็มจาก root
# from mypackage.utils import format_number
# from mypackage.models.user import User

# Relative import - path สัมพันธ์กับ module ปัจจุบัน
# ใช้ได้เฉพาะใน package

# ใน mypackage/core.py:
# from .utils import helper_function    # ← sibling module (. = current package)
# from ..config import settings         # ← parent package (.. = parent)
# from .models.user import User         # ← subpackage

# ตัวอย่าง relative imports
# mypackage/core.py
# from . import utils                   # import sibling module
# from .utils import format_number      # import from sibling
# from ..shared import constants        # import from parent

# Best practices:
# - ใช้ absolute imports ใน script ที่รันตรง
# - ใช้ relative imports ใน package files
# - หลีกเลี่ยง from x import * ยกเว้นใน __init__.py

print("Absolute vs Relative imports:")
print("Absolute: from mypackage.utils import format_number")
print("Relative (ใน package): from .utils import format_number")
```

---

## 7. Standard Library Modules สำคัญ

### 7.1 os module

```python
import os

# Working directory
print(f"CWD: {os.getcwd()}")

# Environment variables
path = os.environ.get("PATH", "")
home = os.environ.get("HOME", "")
print(f"HOME: {home}")

# File operations
print(os.listdir("."))  # list files in current dir

# os.path (pathlib แนะนำกว่าแต่ os.path ยังใช้มาก)
filepath = os.path.join("dir", "subdir", "file.txt")
print(filepath)

print(os.path.exists("."))
print(os.path.basename("/home/user/file.txt"))   # file.txt
print(os.path.dirname("/home/user/file.txt"))    # /home/user
print(os.path.splitext("file.txt.gz"))           # ('file.txt', '.gz')
print(os.path.abspath("file.txt"))

# Process info
print(f"PID: {os.getpid()}")
print(f"CPU count: {os.cpu_count()}")

# os.walk - recursive directory traversal
for root, dirs, files in os.walk("."):
    depth = root.replace(".", "").count(os.sep)
    if depth > 1:
        break  # หยุดที่ depth 1
    print(f"Dir: {root}")
    for f in files[:3]:  # แสดงไม่เกิน 3 ไฟล์
        print(f"  {f}")
```

### 7.2 sys module

```python
import sys

print(f"Python version: {sys.version}")
print(f"Platform: {sys.platform}")
print(f"Executable: {sys.executable}")
print(f"Max int: {sys.maxsize}")

# Arguments
print(f"sys.argv: {sys.argv}")  # [script_name, arg1, arg2, ...]

# stdin/stdout/stderr
print("Hello", file=sys.stderr)  # เขียนไปยัง stderr

# Exit program
# sys.exit(0)   # 0 = success
# sys.exit(1)   # non-zero = error

# Recursion limit
print(f"Recursion limit: {sys.getrecursionlimit()}")
sys.setrecursionlimit(5000)  # เพิ่มถ้าจำเป็น

# Object size
import sys
x = [1, 2, 3, 4, 5]
print(f"Size of x: {sys.getsizeof(x)} bytes")
```

### 7.3 datetime module

```python
from datetime import datetime, date, time, timedelta, timezone

# date - วันที่
today = date.today()
print(f"วันนี้: {today}")
print(f"ปี: {today.year}, เดือน: {today.month}, วัน: {today.day}")

# datetime - วันที่และเวลา
now = datetime.now()
print(f"ตอนนี้: {now}")
print(f"ISO format: {now.isoformat()}")

# สร้างด้วย arguments
birthday = datetime(1990, 5, 15, 10, 30, 0)
print(f"วันเกิด: {birthday}")

# Formatting
print(now.strftime("%Y-%m-%d %H:%M:%S"))
print(now.strftime("%d/%m/%Y %I:%M %p"))  # 15/01/2024 10:30 AM
print(now.strftime("%A, %B %d, %Y"))      # Monday, January 15, 2024

# Parsing
date_str = "2024-01-15 10:30:00"
parsed = datetime.strptime(date_str, "%Y-%m-%d %H:%M:%S")
print(f"Parsed: {parsed}")

# timedelta - ช่วงเวลา
delta = timedelta(days=30, hours=5, minutes=30)
future = now + delta
past = now - timedelta(weeks=2)

print(f"30 วันข้างหน้า: {future.date()}")
print(f"2 สัปดาห์ที่แล้ว: {past.date()}")

# คำนวณอายุ
birth_date = date(1990, 5, 15)
age = (today - birth_date).days // 365
print(f"อายุ: {age} ปี")

# Timezone
utc = timezone.utc
bkk = timezone(timedelta(hours=7))  # UTC+7

utc_now = datetime.now(utc)
bkk_now = utc_now.astimezone(bkk)
print(f"UTC: {utc_now}")
print(f"BKK: {bkk_now}")

# คำนวณวันทำงาน
def count_workdays(start_date, end_date):
    """นับวันทำงาน (จันทร์-ศุกร์)"""
    days = 0
    current = start_date
    while current <= end_date:
        if current.weekday() < 5:  # 0-4 = จันทร์-ศุกร์
            days += 1
        current += timedelta(days=1)
    return days

start = date(2024, 1, 1)
end = date(2024, 1, 31)
print(f"วันทำงานใน Jan 2024: {count_workdays(start, end)} วัน")
```

### 7.4 collections module

```python
from collections import (
    Counter, defaultdict, OrderedDict, 
    deque, namedtuple, ChainMap
)

# Counter - นับความถี่
words = "the quick brown fox jumps over the lazy dog".split()
word_count = Counter(words)
print("Most common:", word_count.most_common(3))

# deque - double-ended queue (เร็วกว่า list สำหรับ prepend/append)
dq = deque([1, 2, 3])
dq.appendleft(0)    # เพิ่มซ้าย O(1)
dq.append(4)        # เพิ่มขวา O(1)
print(dq)           # deque([0, 1, 2, 3, 4])

left = dq.popleft()  # ลบซ้าย O(1)
right = dq.pop()     # ลบขวา O(1)
print(f"ลบ: left={left}, right={right}")
print(dq)           # deque([1, 2, 3])

# deque กับ maxlen
history = deque(maxlen=5)  # เก็บแค่ 5 items ล่าสุด
for i in range(10):
    history.append(i)
print(history)  # deque([5, 6, 7, 8, 9], maxlen=5)

# deque.rotate()
dq = deque([1, 2, 3, 4, 5])
dq.rotate(2)   # หมุนขวา 2
print(dq)      # deque([4, 5, 1, 2, 3])
dq.rotate(-2)  # หมุนซ้าย 2
print(dq)      # deque([1, 2, 3, 4, 5])

# ChainMap - lookup หลาย dict
defaults = {"color": "blue", "size": "medium", "weight": "light"}
user_prefs = {"color": "red"}
env_vars = {"size": "large"}

combined = ChainMap(user_prefs, env_vars, defaults)
print(combined["color"])   # red (จาก user_prefs)
print(combined["size"])    # large (จาก env_vars)
print(combined["weight"])  # light (จาก defaults)
```

### 7.5 pathlib module

```python
from pathlib import Path

# ครอบคลุมใน Part 012 แล้ว
# ตัวอย่างเพิ่มเติม

p = Path.cwd()
print(f"CWD: {p}")

# สร้าง path object
config = p / "config" / "settings.json"
print(f"Config: {config}")
print(f"Exists: {config.exists()}")

# glob patterns
py_files = list(p.glob("**/*.py"))
print(f"Python files: {len(py_files)}")

# Path methods
for part in p.parts:
    print(f"  Part: {part}")
```

### 7.6 random module

```python
import random

# รับตัวเลขสุ่ม
print(random.random())          # float 0.0-1.0
print(random.randint(1, 100))   # int 1-100
print(random.uniform(0, 1))     # float 0-1
print(random.randrange(0, 10, 2))  # เลขคู่ 0-8

# สุ่มจาก sequence
fruits = ["apple", "banana", "cherry", "date"]
print(random.choice(fruits))     # สุ่ม 1 ตัว
print(random.choices(fruits, k=3))  # สุ่มพร้อม replacement
print(random.sample(fruits, 3))  # สุ่มไม่ซ้ำ 3 ตัว

# สุ่มด้วย weights
items = ["common", "uncommon", "rare", "legendary"]
weights = [60, 25, 10, 5]
drops = random.choices(items, weights=weights, k=10)
print(Counter(drops))

# Shuffle
cards = list(range(1, 53))
random.shuffle(cards)
print(cards[:10])

# Seed - reproducible random
random.seed(42)
print([random.randint(1, 10) for _ in range(5)])  # เสมอเดิม

random.seed(42)
print([random.randint(1, 10) for _ in range(5)])  # เหมือนกัน
```

### 7.7 itertools module

```python
import itertools

# count() - นับไม่หยุด
for i, x in enumerate(itertools.count(10, 2)):  # เริ่ม 10, step 2
    if i >= 5:
        break
    print(x, end=" ")  # 10 12 14 16 18
print()

# cycle() - วนซ้ำ
colors = itertools.cycle(["red", "green", "blue"])
for _, color in zip(range(7), colors):
    print(color, end=" ")  # red green blue red green blue red
print()

# chain() - ต่อ iterables
a = [1, 2, 3]
b = [4, 5, 6]
c = [7, 8, 9]
print(list(itertools.chain(a, b, c)))  # [1,...,9]

# combinations() - การรวมกัน
items = ["A", "B", "C", "D"]
print(list(itertools.combinations(items, 2)))
# [('A','B'), ('A','C'), ('A','D'), ('B','C'), ('B','D'), ('C','D')]

# permutations() - การเรียงสับเปลี่ยน
print(list(itertools.permutations([1, 2, 3])))
# [(1,2,3), (1,3,2), (2,1,3), (2,3,1), (3,1,2), (3,2,1)]

# product() - Cartesian product
print(list(itertools.product([0, 1], repeat=3)))
# [(0,0,0), (0,0,1), (0,1,0), (0,1,1), ...]

# groupby() - จัดกลุ่ม
data = sorted([("A", 1), ("A", 2), ("B", 3), ("B", 4), ("C", 5)], key=lambda x: x[0])
for key, group in itertools.groupby(data, key=lambda x: x[0]):
    values = [v for _, v in group]
    print(f"{key}: {values}")

# islice() - slice ของ iterator
gen = (x**2 for x in range(100))
first_5 = list(itertools.islice(gen, 5))
print(first_5)  # [0, 1, 4, 9, 16]

next_5 = list(itertools.islice(gen, 5))
print(next_5)   # [25, 36, 49, 64, 81]
```

### 7.8 functools module

```python
from functools import reduce, partial, lru_cache, wraps, cached_property

# reduce() - fold/accumulate
numbers = [1, 2, 3, 4, 5]
total = reduce(lambda a, b: a + b, numbers)
print(total)  # 15

product = reduce(lambda a, b: a * b, numbers)
print(product)  # 120

# partial() - สร้างฟังก์ชันใหม่จากฟังก์ชันเดิมพร้อม args บางตัว
def power(base, exp):
    return base ** exp

square = partial(power, exp=2)
cube = partial(power, exp=3)

print(square(5))  # 25
print(cube(3))    # 27

# lru_cache() - memoization
@lru_cache(maxsize=None)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

print(fibonacci(50))  # เร็วมากเพราะ cache

# ดู cache info
print(fibonacci.cache_info())  # CacheInfo(hits=..., misses=..., ...)

# cached_property - property ที่ cache ผลลัพธ์
class DataProcessor:
    def __init__(self, data):
        self._data = data
    
    @cached_property
    def processed_data(self):
        print("กำลัง process... (รันครั้งเดียว)")
        return [x * 2 for x in self._data]
    
    @cached_property
    def statistics(self):
        print("กำลังคำนวณ stats... (รันครั้งเดียว)")
        data = self.processed_data
        return {"sum": sum(data), "avg": sum(data)/len(data)}

dp = DataProcessor([1, 2, 3, 4, 5])
print(dp.processed_data)   # process ครั้งแรก
print(dp.processed_data)   # ใช้ cache
print(dp.statistics)       # คำนวณครั้งแรก
print(dp.statistics)       # ใช้ cache

# wraps() - รักษา metadata ของ function ที่ decorate
def my_decorator(func):
    @wraps(func)  # ← สำคัญ! รักษา __name__, __doc__
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")
        return func(*args, **kwargs)
    return wrapper

@my_decorator
def greet(name):
    """ฟังก์ชันทักทาย"""
    return f"Hello, {name}!"

print(greet("Alice"))
print(greet.__name__)  # greet (ไม่ใช่ wrapper)
print(greet.__doc__)   # ฟังก์ชันทักทาย
```

---

## 8. ตัวอย่างโปรแกรมจริง: Plugin System

```python
"""
ระบบ Plugin ที่ dynamic load modules
"""
import importlib
import inspect
from abc import ABC, abstractmethod
from pathlib import Path
from typing import Dict, List, Type, Any
import sys
import tempfile

# Base plugin interface
class Plugin(ABC):
    @property
    @abstractmethod
    def name(self) -> str:
        pass
    
    @property
    @abstractmethod
    def version(self) -> str:
        pass
    
    @property
    @abstractmethod
    def description(self) -> str:
        pass
    
    @abstractmethod
    def execute(self, data: Any) -> Any:
        pass

class PluginManager:
    def __init__(self):
        self._plugins: Dict[str, Plugin] = {}
    
    def register(self, plugin: Plugin) -> None:
        """ลงทะเบียน plugin"""
        if plugin.name in self._plugins:
            print(f"แทนที่ plugin เก่า: {plugin.name}")
        self._plugins[plugin.name] = plugin
        print(f"ลงทะเบียน plugin: {plugin.name} v{plugin.version}")
    
    def load_from_module(self, module_name: str) -> int:
        """โหลด plugins จาก module"""
        try:
            module = importlib.import_module(module_name)
        except ImportError as e:
            print(f"ไม่สามารถโหลด {module_name}: {e}")
            return 0
        
        count = 0
        for name, obj in inspect.getmembers(module, inspect.isclass):
            if (issubclass(obj, Plugin) and 
                obj is not Plugin and 
                not inspect.isabstract(obj)):
                try:
                    instance = obj()
                    self.register(instance)
                    count += 1
                except Exception as e:
                    print(f"ไม่สามารถสร้าง {name}: {e}")
        
        return count
    
    def get(self, name: str) -> Plugin:
        if name not in self._plugins:
            raise KeyError(f"ไม่พบ plugin: {name}")
        return self._plugins[name]
    
    def execute(self, plugin_name: str, data: Any) -> Any:
        return self.get(plugin_name).execute(data)
    
    def list_plugins(self) -> List[dict]:
        return [
            {"name": p.name, "version": p.version, "description": p.description}
            for p in self._plugins.values()
        ]

# สร้าง plugins จริง
class UpperCasePlugin(Plugin):
    @property
    def name(self): return "uppercase"
    @property
    def version(self): return "1.0.0"
    @property
    def description(self): return "แปลง text เป็นตัวพิมพ์ใหญ่"
    def execute(self, data):
        return str(data).upper()

class ReversePlugin(Plugin):
    @property
    def name(self): return "reverse"
    @property
    def version(self): return "1.0.0"
    @property
    def description(self): return "กลับลำดับ text"
    def execute(self, data):
        return str(data)[::-1]

class WordCountPlugin(Plugin):
    @property
    def name(self): return "word_count"
    @property
    def version(self): return "1.1.0"
    @property
    def description(self): return "นับจำนวนคำใน text"
    def execute(self, data):
        words = str(data).split()
        return {"words": len(words), "chars": len(str(data))}

# ทดสอบ
pm = PluginManager()
pm.register(UpperCasePlugin())
pm.register(ReversePlugin())
pm.register(WordCountPlugin())

print("\n=== Plugins ===")
for p in pm.list_plugins():
    print(f"  {p['name']} v{p['version']}: {p['description']}")

text = "Hello World Python Programming"
print(f"\nInput: {text!r}")
print(f"uppercase: {pm.execute('uppercase', text)}")
print(f"reverse: {pm.execute('reverse', text)}")
print(f"word_count: {pm.execute('word_count', text)}")
```

---

## 9. Exercises

### Exercise 1: Module Inspector

```python
"""
สร้างเครื่องมือ inspect modules:
1. แสดง functions ทั้งหมดใน module
2. แสดง classes และ methods
3. สร้าง documentation summary
"""
import inspect
import math
import os
from typing import Any

def inspect_module(module: Any) -> dict:
    """วิเคราะห์ module และคืนข้อมูล"""
    info = {
        "name": module.__name__,
        "file": getattr(module, "__file__", "built-in"),
        "doc": (module.__doc__ or "").strip()[:100],
        "functions": [],
        "classes": [],
        "constants": [],
    }
    
    for name, obj in sorted(inspect.getmembers(module)):
        if name.startswith("_"):
            continue
        
        if inspect.isfunction(obj) or inspect.isbuiltin(obj):
            doc = (getattr(obj, "__doc__", "") or "").split("\n")[0].strip()
            info["functions"].append({"name": name, "doc": doc[:60]})
        
        elif inspect.isclass(obj):
            methods = [
                m for m in dir(obj)
                if not m.startswith("_") and callable(getattr(obj, m))
            ]
            info["classes"].append({
                "name": name,
                "methods": methods[:5],
                "doc": (obj.__doc__ or "").split("\n")[0].strip()[:60],
            })
        
        elif isinstance(obj, (int, float, str)) and name.isupper():
            info["constants"].append({"name": name, "value": obj})
    
    return info

# วิเคราะห์ math module
info = inspect_module(math)
print(f"Module: {info['name']}")
print(f"Constants ({len(info['constants'])}): {[c['name'] for c in info['constants']]}")
print(f"\nFunctions ({len(info['functions'])}):")
for f in info["functions"][:8]:
    print(f"  {f['name']:20}: {f['doc']}")

# วิเคราะห์ os module
os_info = inspect_module(os)
print(f"\n{os_info['name']}: {len(os_info['functions'])} functions, {len(os_info['classes'])} classes")
```

### Exercise 2: Configuration Manager

```python
"""
Config Manager ที่ใช้ Pattern ต่างๆ:
1. Singleton pattern
2. Config จาก environment variables
3. Config จากไฟล์
4. Type conversion
"""
import os
import json
from pathlib import Path
from typing import Any, Optional

class Config:
    """Singleton Config Manager"""
    _instance = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._data = {}
        return cls._instance
    
    def set(self, key: str, value: Any) -> None:
        self._data[key] = value
    
    def get(self, key: str, default: Any = None, type_: type = None) -> Any:
        value = self._data.get(key, os.environ.get(key, default))
        if value is not None and type_ is not None:
            try:
                if type_ == bool:
                    value = str(value).lower() in ("true", "1", "yes")
                else:
                    value = type_(value)
            except (ValueError, TypeError):
                return default
        return value
    
    def get_int(self, key: str, default: int = 0) -> int:
        return self.get(key, default, int)
    
    def get_bool(self, key: str, default: bool = False) -> bool:
        return self.get(key, default, bool)
    
    def get_list(self, key: str, separator: str = ",") -> list:
        value = self.get(key, "")
        return [v.strip() for v in str(value).split(separator) if v.strip()]
    
    def load_json(self, filepath: str) -> None:
        path = Path(filepath)
        if path.exists():
            with open(path, "r", encoding="utf-8") as f:
                data = json.load(f)
            self._data.update(data)

# ทดสอบ
config = Config()

# ตั้งค่า
config.set("DATABASE_HOST", "localhost")
config.set("DATABASE_PORT", "5432")
config.set("DEBUG", "true")
config.set("ALLOWED_HOSTS", "localhost,127.0.0.1,0.0.0.0")
config.set("MAX_CONNECTIONS", "10")

print("=== Config Values ===")
print(f"DB Host: {config.get('DATABASE_HOST')}")
print(f"DB Port: {config.get_int('DATABASE_PORT')}")
print(f"Debug: {config.get_bool('DEBUG')}")
print(f"Hosts: {config.get_list('ALLOWED_HOSTS')}")
print(f"Max Conn: {config.get_int('MAX_CONNECTIONS')}")
print(f"Missing (default): {config.get('MISSING_KEY', 'DEFAULT_VALUE')}")

# Singleton - same instance
config2 = Config()
print(f"\nSingleton: {config is config2}")  # True
```

---

## 10. สรุป Part 014

### สิ่งที่เรียนรู้:

✅ **Module** - ไฟล์ Python ที่ reuse ได้  
✅ **import** - 6 รูปแบบของ import  
✅ **sys.path** - ตำแหน่งที่ Python ค้นหา modules  
✅ **\_\_name\_\_** - รู้ว่ากำลัง run หรือ import  
✅ **Package** - directory + \_\_init\_\_.py  
✅ **\_\_init\_\_.py** - กำหนด public API  
✅ **Relative imports** - ใช้ใน package (.)  
✅ **os module** - filesystem, process, env vars  
✅ **sys module** - Python runtime info  
✅ **datetime module** - date/time operations  
✅ **collections** - Counter, deque, defaultdict, ChainMap  
✅ **itertools** - chain, product, groupby, combinations  
✅ **functools** - reduce, partial, lru_cache, wraps  

### Quick Reference:

```python
# Import styles
import module
import module as alias
from module import name
from module import name as alias
from . import sibling           # relative (in package)
from .sibling import func      # relative

# Package structure
# pkg/
# ├── __init__.py
# ├── module.py
# └── sub/
#     ├── __init__.py
#     └── mod.py

# __init__.py
__version__ = "1.0"
from .module import PublicClass
__all__ = ["PublicClass"]

# __name__
if __name__ == "__main__":
    main()  # รันเฉพาะเมื่อเรียกตรง

# Useful modules
import os, sys, math
from datetime import datetime, date, timedelta
from collections import Counter, defaultdict, deque
from functools import lru_cache, partial, reduce
from itertools import chain, product, groupby
from pathlib import Path
```

---

## ➡️ ถัดไป: Part 015 - OOP พื้นฐาน

ใน Part ถัดไป เราจะเรียนรู้:
- class definition และ \_\_init\_\_
- Instance/Class/Static methods
- Properties และ Getters/Setters
- Magic methods (\_\_str\_\_, \_\_repr\_\_, \_\_add\_\_, \_\_len\_\_, \_\_eq\_\_)
- Inheritance และ Polymorphism

---

*Part 014/100+ | Python Course - Beginner to World-Class*
