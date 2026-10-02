# Part 014: Modules and Packages
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ import, from import ได้
- สร้าง Package ด้วย `__init__.py` ได้
- เข้าใจ `__name__ == "__main__"` ได้
- จัดการ `sys.path` ได้
- ใช้ pip และ virtual environments ได้
- สร้างและใช้ `requirements.txt` ได้

---

## 1. Modules พื้นฐาน

Module คือไฟล์ `.py` ที่มี Python code อยู่ข้างใน ใช้แบ่งโค้ดเป็นส่วนๆ

```python
# ===== import แบบต่างๆ =====

# import ทั้ง module
import math
print(math.pi)         # 3.14159...
print(math.sqrt(16))   # 4.0
print(math.floor(3.7)) # 3

# from import - import เฉพาะที่ต้องการ
from math import pi, sqrt, ceil, floor
print(pi)          # ใช้โดยตรงได้เลย
print(sqrt(25))    # 5.0

# import พร้อม alias
import math as m
from math import pi as PI
print(m.e)   # 2.718...
print(PI)    # 3.14159...

# import ทุกอย่าง (ไม่แนะนำ!)
# from math import *
# print(sin(pi/2))  # ใช้ได้แต่ทำให้ namespace ยุ่ง

# ===== Standard Library Modules =====

# os - ระบบปฏิบัติการ
import os
print(f"CWD: {os.getcwd()}")
print(f"Home: {os.path.expanduser('~')}")

# sys - ข้อมูล Python interpreter
import sys
print(f"Python version: {sys.version}")
print(f"Platform: {sys.platform}")

# datetime - วันที่และเวลา
from datetime import datetime, date, timedelta
now = datetime.now()
today = date.today()
tomorrow = today + timedelta(days=1)
print(f"วันนี้: {today}")
print(f"พรุ่งนี้: {tomorrow}")

# random - สุ่มตัวเลข
import random
print(random.randint(1, 100))
print(random.choice(["apple", "banana", "cherry"]))
random.shuffle([1, 2, 3, 4, 5])

# collections - data structures พิเศษ
from collections import Counter, defaultdict, OrderedDict, namedtuple

# itertools - เครื่องมือสำหรับ iteration
import itertools
combinations = list(itertools.combinations([1, 2, 3], 2))
print(f"Combinations: {combinations}")

# functools - เครื่องมือสำหรับ functions
from functools import reduce, partial
total = reduce(lambda a, b: a + b, [1, 2, 3, 4, 5])
print(f"Sum: {total}")  # 15
```

## 2. สร้าง Module เอง

```python
# ===== สร้างไฟล์ utils.py =====
# บันทึกเป็น /tmp/myproject/utils.py

utils_content = '''"""
Utility functions สำหรับโปรเจกต์
"""

def greet(name: str, greeting: str = "สวัสดี") -> str:
    """สร้างคำทักทาย"""
    return f"{greeting}, {name}!"

def validate_email(email: str) -> bool:
    """ตรวจสอบ email address"""
    import re
    pattern = r"^[\\w.+-]+@[\\w-]+\\.[\\w.]+$"
    return bool(re.match(pattern, email))

def format_number(number: float, decimal: int = 2) -> str:
    """Format ตัวเลขพร้อม comma separator"""
    return f"{number:,.{decimal}f}"

PI = 3.14159265358979
VERSION = "1.0.0"

class Calculator:
    """Simple calculator"""
    
    def add(self, a, b):
        return a + b
    
    def subtract(self, a, b):
        return a - b
    
    def multiply(self, a, b):
        return a * b
    
    def divide(self, a, b):
        if b == 0:
            raise ValueError("หารด้วยศูนย์ไม่ได้")
        return a / b

if __name__ == "__main__":
    # ทดสอบเมื่อ run โดยตรง
    print("Testing utils...")
    print(greet("World"))
    print(validate_email("test@example.com"))
    print(format_number(1234567.89))
'''

import os
os.makedirs("/tmp/myproject", exist_ok=True)
with open("/tmp/myproject/utils.py", "w", encoding="utf-8") as f:
    f.write(utils_content)
print("สร้าง utils.py แล้ว")

# ===== ใช้งาน module =====
import sys
sys.path.insert(0, "/tmp/myproject")  # เพิ่ม path

import utils

# ใช้ functions
print(utils.greet("สมชาย"))
print(utils.validate_email("test@example.com"))
print(utils.format_number(1234567.89))

# ใช้ constants
print(utils.PI)
print(utils.VERSION)

# ใช้ class
calc = utils.Calculator()
print(calc.add(10, 5))
```

## 3. `__name__` == `"__main__"`

```python
# ===== เข้าใจ __name__ =====
# เมื่อ run ไฟล์โดยตรง:  __name__ == "__main__"
# เมื่อ import ไฟล์นั้น:  __name__ == "ชื่อ module"

# ตัวอย่าง: สร้าง calculator.py
calc_content = '''"""
Calculator module
"""

def add(a: float, b: float) -> float:
    return a + b

def subtract(a: float, b: float) -> float:
    return a - b

def multiply(a: float, b: float) -> float:
    return a * b

def divide(a: float, b: float) -> float:
    if b == 0:
        raise ZeroDivisionError("หารด้วยศูนย์ไม่ได้")
    return a / b

def demo():
    """Demo function"""
    print("=== Calculator Demo ===")
    print(f"10 + 5 = {add(10, 5)}")
    print(f"10 - 5 = {subtract(10, 5)}")
    print(f"10 * 5 = {multiply(10, 5)}")
    print(f"10 / 5 = {divide(10, 5)}")

if __name__ == "__main__":
    # โค้ดนี้จะทำงานเฉพาะเมื่อ run ไฟล์นี้โดยตรง
    # ถ้า import จาก module อื่น จะไม่ทำงาน
    print(f"Running: {__name__}")
    demo()
'''

with open("/tmp/myproject/calculator.py", "w", encoding="utf-8") as f:
    f.write(calc_content)

# เมื่อ import
import importlib.util
spec = importlib.util.spec_from_file_location("calculator", 
                                               "/tmp/myproject/calculator.py")
calculator = importlib.util.module_from_spec(spec)
spec.loader.exec_module(calculator)

# ใช้งาน functions
print(f"3 + 4 = {calculator.add(3, 4)}")
print(f"__name__ ของ module: {calculator.__name__}")

# ===== ทำไม if __name__ == "__main__" ถึงสำคัญ =====
"""
ประโยชน์:
1. ป้องกัน code ทำงานเมื่อถูก import
2. ทำให้ module ทดสอบได้โดยตรง
3. แยก test code ออกจาก module code
4. เป็น entry point ของโปรแกรม

ตัวอย่างรูปแบบมาตรฐาน:
"""

# main.py
main_content = '''#!/usr/bin/env python3
"""Main entry point"""

from calculator import add, multiply
from utils import greet, format_number

def main():
    """Main function"""
    print(greet("World", "Hello"))
    
    result = add(100, 200)
    print(f"100 + 200 = {format_number(result, 0)}")
    
    product = multiply(1234, 5678)
    print(f"1234 * 5678 = {format_number(product, 0)}")

if __name__ == "__main__":
    main()
'''

with open("/tmp/myproject/main.py", "w", encoding="utf-8") as f:
    f.write(main_content)
print("สร้าง main.py แล้ว")
```

## 4. Packages - สร้าง Package เอง

```python
# ===== โครงสร้าง Package =====
"""
mypackage/
├── __init__.py          ← ทำให้ folder เป็น package
├── core.py
├── utils.py
└── models/
    ├── __init__.py
    ├── user.py
    └── product.py
"""

import os

# สร้าง package structure
package_dir = "/tmp/mypackage"
models_dir = f"{package_dir}/models"
os.makedirs(models_dir, exist_ok=True)

# ===== __init__.py ของ package หลัก =====
init_content = '''"""
mypackage - ชุดเครื่องมือสำหรับแอปพลิเคชัน
"""

# กำหนด public API ของ package
from .core import Config, Database
from .utils import format_date, slugify

# version
__version__ = "1.0.0"
__author__ = "สมชาย ใจดี"

# ควบคุม what gets imported with "from mypackage import *"
__all__ = ["Config", "Database", "format_date", "slugify"]

print(f"mypackage {__version__} loaded")
'''

with open(f"{package_dir}/__init__.py", "w") as f:
    f.write(init_content)

# ===== core.py =====
core_content = '''"""
Core classes ของ package
"""

class Config:
    """Application configuration"""
    
    def __init__(self, **kwargs):
        self._data = {
            "debug": False,
            "database_url": "sqlite:///app.db",
            "secret_key": "change-this-in-production",
            **kwargs
        }
    
    def get(self, key: str, default=None):
        return self._data.get(key, default)
    
    def set(self, key: str, value) -> None:
        self._data[key] = value
    
    def __repr__(self):
        keys = list(self._data.keys())
        return f"Config({keys})"

class Database:
    """Database connection manager"""
    
    def __init__(self, url: str):
        self.url = url
        self._connected = False
    
    def connect(self) -> bool:
        print(f"เชื่อมต่อ: {self.url}")
        self._connected = True
        return True
    
    def disconnect(self) -> None:
        self._connected = False
        print("ตัดการเชื่อมต่อแล้ว")
    
    @property
    def is_connected(self) -> bool:
        return self._connected
'''

with open(f"{package_dir}/core.py", "w") as f:
    f.write(core_content)

# ===== utils.py =====
utils_content = '''"""
Utility functions
"""

import re
from datetime import datetime

def format_date(dt: datetime, fmt: str = "%d/%m/%Y") -> str:
    """Format datetime object"""
    return dt.strftime(fmt)

def slugify(text: str) -> str:
    """แปลง text เป็น URL-friendly slug"""
    text = text.lower().strip()
    text = re.sub(r"[^\\w\\s-]", "", text)
    text = re.sub(r"[\\s_-]+", "-", text)
    text = re.sub(r"^-+|-+$", "", text)
    return text
'''

with open(f"{package_dir}/utils.py", "w") as f:
    f.write(utils_content)

# ===== models/__init__.py =====
models_init = '''"""Models package"""
from .user import User
from .product import Product

__all__ = ["User", "Product"]
'''

with open(f"{models_dir}/__init__.py", "w") as f:
    f.write(models_init)

# ===== models/user.py =====
user_content = '''"""User model"""

class User:
    def __init__(self, id: int, name: str, email: str):
        self.id = id
        self.name = name
        self.email = email
    
    def __repr__(self):
        return f"User(id={self.id}, name={self.name!r})"
    
    def to_dict(self) -> dict:
        return {"id": self.id, "name": self.name, "email": self.email}
'''

with open(f"{models_dir}/user.py", "w") as f:
    f.write(user_content)

# ===== models/product.py =====
product_content = '''"""Product model"""

class Product:
    def __init__(self, id: str, name: str, price: float):
        self.id = id
        self.name = name
        self.price = price
    
    def __repr__(self):
        return f"Product(id={self.id!r}, name={self.name!r}, price={self.price})"
    
    def to_dict(self) -> dict:
        return {"id": self.id, "name": self.name, "price": self.price}
'''

with open(f"{models_dir}/product.py", "w") as f:
    f.write(product_content)

print("สร้าง package structure แล้ว!")
print("Structure:")
for root, dirs, files in os.walk(package_dir):
    level = root.replace(package_dir, "").count(os.sep)
    indent = "  " * level
    print(f"{indent}{os.path.basename(root)}/")
    for file in files:
        print(f"{indent}  {file}")
```

## 5. ใช้งาน Package

```python
# เพิ่ม path เพื่อ import package
import sys
sys.path.insert(0, "/tmp")

# ===== import จาก package =====

# import ทั้ง package
import mypackage

# import specific items
from mypackage import Config, Database
from mypackage.utils import format_date, slugify
from mypackage.models import User, Product
from mypackage.models.user import User as UserModel

# ===== ใช้งาน =====
config = Config(debug=True, database_url="postgresql://localhost/mydb")
print(f"Config: {config}")
print(f"Debug: {config.get('debug')}")

db = Database(config.get("database_url"))
db.connect()
print(f"Connected: {db.is_connected}")
db.disconnect()

user = User(1, "สมชาย ใจดี", "somchai@example.com")
print(f"User: {user}")
print(f"Dict: {user.to_dict()}")

product = Product("P001", "Python Book", 599.00)
print(f"Product: {product}")

from datetime import datetime
formatted = format_date(datetime.now(), "%d %B %Y")
print(f"Date: {formatted}")

slug = slugify("Hello World Python! 123")
print(f"Slug: {slug}")

# ===== Relative imports (ใช้ภายใน package) =====
"""
ภายใน package ใช้ relative imports:
from . import module          # import module ใน package เดียวกัน
from .module import thing     # import จาก module ใน package เดียวกัน
from ..module import thing    # import จาก parent package
from ..sibling import thing   # import จาก sibling package
"""
```

## 6. sys.path

```python
import sys

# ===== ดู Python path =====
print("Python search paths:")
for i, path in enumerate(sys.path):
    print(f"  {i}: {path}")

# ===== เพิ่ม path =====
# วิธีที่ 1: sys.path.insert/append
sys.path.insert(0, "/tmp/myproject")  # เพิ่มไว้ต้น (สำคัญที่สุด)
sys.path.append("/home/user/libs")     # เพิ่มไว้ท้าย

# วิธีที่ 2: PYTHONPATH environment variable
# export PYTHONPATH=/tmp/myproject:$PYTHONPATH

# วิธีที่ 3: .pth file ใน site-packages
# /usr/lib/python3.x/site-packages/mylibs.pth
# ข้างในไฟล์: /path/to/my/libs

# ===== ค้นหา module location =====
import os
module_location = os.path.dirname(os.__file__)
print(f"\nos module location: {module_location}")

import json
print(f"json module: {json.__file__}")

# ===== importlib - dynamic imports =====
import importlib

# import module แบบ dynamic
module_name = "math"
math_module = importlib.import_module(module_name)
print(f"\nDynamic import: {math_module.sqrt(16)}")

# reload module หลังแก้ไข
# importlib.reload(my_module)

# ===== ตรวจสอบว่า module มีอยู่ไหม =====
from importlib.util import find_spec

def is_module_available(module_name: str) -> bool:
    return find_spec(module_name) is not None

print(f"\njson available: {is_module_available('json')}")
print(f"requests available: {is_module_available('requests')}")
print(f"django available: {is_module_available('django')}")
```

## 7. pip - Python Package Manager

```bash
# ===== คำสั่ง pip พื้นฐาน =====

# ติดตั้ง package
pip install requests
pip install django==4.2
pip install flask>=2.0

# ติดตั้งหลาย packages พร้อมกัน
pip install requests flask sqlalchemy

# อัพเดต package
pip install --upgrade requests
pip install --upgrade pip  # อัพเดต pip เอง

# ถอนการติดตั้ง
pip uninstall requests
pip uninstall -y requests  # ไม่ถาม yes/no

# ดู packages ที่ติดตั้ง
pip list
pip list --outdated  # packages ที่มีเวอร์ชันใหม่

# ดูข้อมูล package
pip show requests
pip show django

# ค้นหา packages
pip search keyword

# ดู dependencies
pip install pipdeptree
pipdeptree

# สร้าง requirements.txt
pip freeze > requirements.txt

# ติดตั้งจาก requirements.txt
pip install -r requirements.txt

# ===== ติดตั้งใน mode development =====
# pip install -e .  # install ในโหมด editable

# ===== ดาวน์โหลดโดยไม่ติดตั้ง =====
# pip download requests -d ./packages/
```

## 8. Virtual Environments

```bash
# ===== ทำไมต้องใช้ Virtual Environment? =====
# - แยก dependencies ของแต่ละโปรเจกต์
# - ป้องกัน version conflicts
# - Reproducible environments

# ===== venv (built-in) =====

# สร้าง virtual environment
python3 -m venv venv          # สร้างใน folder "venv"
python3 -m venv .venv         # ซ่อน folder ด้วย .
python3 -m venv /path/to/venv # กำหนด path เอง

# Activate (Linux/Mac)
source venv/bin/activate

# Activate (Windows)
venv\\Scripts\\activate
venv\\Scripts\\activate.bat  # CMD
venv\\Scripts\\Activate.ps1  # PowerShell

# Deactivate
deactivate

# ลบ virtual environment
rm -rf venv

# ===== ขั้นตอนการตั้ง project ใหม่ =====
mkdir myproject
cd myproject

python3 -m venv venv
source venv/bin/activate

pip install requests flask sqlalchemy
pip freeze > requirements.txt

# เช็ค environment
which python   # ควรชี้ไป venv
pip list

# ===== pipenv (alternative) =====
pip install pipenv
pipenv install requests
pipenv install pytest --dev  # dev dependency
pipenv shell                  # activate
pipenv run python app.py     # run โดยไม่ต้อง activate

# ===== poetry (modern tool) =====
curl -sSL https://install.python-poetry.org | python3 -
poetry new myproject
poetry add requests flask
poetry add pytest --group dev
poetry install
poetry shell
```

## 9. requirements.txt

```python
# ===== สร้าง requirements.txt =====

# วิธีที่ 1: pip freeze (ทุก dependencies)
# pip freeze > requirements.txt
# ได้:
# certifi==2024.1.1
# charset-normalizer==3.3.2
# idna==3.6
# requests==2.31.0
# urllib3==2.1.0

# วิธีที่ 2: เขียนเอง (แนะนำ สำหรับ production)
requirements_content = """
# Web Framework
Django>=4.2,<5.0
djangorestframework>=3.14

# Database
psycopg2-binary>=2.9
SQLAlchemy>=2.0

# Task Queue
celery>=5.3
redis>=5.0

# Testing (dev only)
pytest>=7.4
pytest-django>=4.7
factory-boy>=3.3

# Utilities
python-dotenv>=1.0
Pillow>=10.0
requests>=2.31
"""

# บันทึก requirements
with open("/tmp/requirements.txt", "w") as f:
    f.write(requirements_content)
print("สร้าง requirements.txt แล้ว")

# ===== แยก environments =====
# requirements/
# ├── base.txt       # ทุก environment
# ├── development.txt # dev เท่านั้น (-r base.txt)
# ├── staging.txt    # staging (-r base.txt)
# └── production.txt # production (-r base.txt)

base_req = """
# Base requirements
Django>=4.2
psycopg2-binary>=2.9
redis>=5.0
python-dotenv>=1.0
"""

dev_req = """
-r base.txt

# Development only
pytest>=7.4
pytest-django>=4.7
django-debug-toolbar>=4.2
black>=23.0
flake8>=6.0
mypy>=1.0
"""

prod_req = """
-r base.txt

# Production only
gunicorn>=21.0
sentry-sdk>=1.39
"""

import os
os.makedirs("/tmp/requirements", exist_ok=True)
for filename, content in [("base.txt", base_req), 
                            ("development.txt", dev_req),
                            ("production.txt", prod_req)]:
    with open(f"/tmp/requirements/{filename}", "w") as f:
        f.write(content)

print("สร้าง requirements files แล้ว")
```

## 10. ตัวอย่างโปรเจกต์: สร้าง Library

```python
"""
สร้าง text processing library ครบวงจร
"""

import os

# ===== โครงสร้างของ library =====
"""
texttools/
├── __init__.py
├── cleaner.py     - ทำความสะอาด text
├── analyzer.py    - วิเคราะห์ text
├── formatter.py   - จัด format text
└── exceptions.py  - custom exceptions
"""

lib_dir = "/tmp/texttools"
os.makedirs(lib_dir, exist_ok=True)

# exceptions.py
with open(f"{lib_dir}/exceptions.py", "w") as f:
    f.write('''"""Custom exceptions"""

class TextToolsError(Exception):
    """Base exception"""
    pass

class EmptyTextError(TextToolsError):
    """Text is empty"""
    pass

class InvalidInputError(TextToolsError):
    """Input type is invalid"""
    pass
''')

# cleaner.py
with open(f"{lib_dir}/cleaner.py", "w") as f:
    f.write('''"""Text cleaning utilities"""
import re
from .exceptions import EmptyTextError, InvalidInputError

def clean(text: str, *, 
          strip: bool = True,
          lowercase: bool = False,
          remove_extra_spaces: bool = True,
          remove_punctuation: bool = False) -> str:
    """ทำความสะอาด text"""
    if not isinstance(text, str):
        raise InvalidInputError(f"ต้องเป็น str ไม่ใช่ {type(text).__name__}")
    
    if strip:
        text = text.strip()
    
    if lowercase:
        text = text.lower()
    
    if remove_extra_spaces:
        text = re.sub(r"\\s+", " ", text)
    
    if remove_punctuation:
        text = re.sub(r"[^\\w\\s]", "", text)
    
    return text

def remove_html_tags(text: str) -> str:
    """ลบ HTML tags"""
    return re.sub(r"<[^>]+>", "", text)

def normalize_whitespace(text: str) -> str:
    """normalize whitespace"""
    return " ".join(text.split())
''')

# analyzer.py
with open(f"{lib_dir}/analyzer.py", "w") as f:
    f.write('''"""Text analysis utilities"""
import re
from collections import Counter
from .exceptions import EmptyTextError

def word_count(text: str) -> int:
    """นับจำนวนคำ"""
    if not text.strip():
        raise EmptyTextError("Text ว่างเปล่า")
    return len(text.split())

def char_count(text: str, include_spaces: bool = True) -> int:
    """นับจำนวนตัวอักษร"""
    if not include_spaces:
        text = text.replace(" ", "")
    return len(text)

def sentence_count(text: str) -> int:
    """นับจำนวนประโยค"""
    sentences = re.split(r"[.!?]+", text)
    return len([s for s in sentences if s.strip()])

def word_frequency(text: str, top_n: int = None) -> dict:
    """นับความถี่ของคำ"""
    words = re.findall(r"\\b\\w+\\b", text.lower())
    freq = Counter(words)
    if top_n:
        return dict(freq.most_common(top_n))
    return dict(freq)

def reading_time(text: str, wpm: int = 200) -> float:
    """ประมาณเวลาอ่าน (นาที)"""
    words = word_count(text)
    return words / wpm
''')

# formatter.py
with open(f"{lib_dir}/formatter.py", "w") as f:
    f.write('''"""Text formatting utilities"""
import textwrap

def wrap(text: str, width: int = 80, indent: str = "") -> str:
    """ตัดบรรทัดตามความกว้าง"""
    return textwrap.fill(text, width=width, initial_indent=indent,
                        subsequent_indent=indent)

def truncate(text: str, max_length: int, suffix: str = "...") -> str:
    """ตัด text ให้สั้นลง"""
    if len(text) <= max_length:
        return text
    return text[:max_length - len(suffix)] + suffix

def title_case(text: str) -> str:
    """แปลงเป็น Title Case"""
    return text.title()

def pad(text: str, width: int, align: str = "left", fill: str = " ") -> str:
    """เพิ่ม padding"""
    if align == "left":
        return text.ljust(width, fill)
    elif align == "right":
        return text.rjust(width, fill)
    else:
        return text.center(width, fill)
''')

# __init__.py
with open(f"{lib_dir}/__init__.py", "w") as f:
    f.write('''"""
texttools - Text Processing Library
"""

from .cleaner import clean, remove_html_tags, normalize_whitespace
from .analyzer import word_count, char_count, word_frequency, reading_time
from .formatter import wrap, truncate, title_case, pad
from .exceptions import TextToolsError, EmptyTextError, InvalidInputError

__version__ = "1.0.0"
__all__ = [
    "clean", "remove_html_tags", "normalize_whitespace",
    "word_count", "char_count", "word_frequency", "reading_time",
    "wrap", "truncate", "title_case", "pad",
    "TextToolsError", "EmptyTextError", "InvalidInputError"
]
''')

# ===== ทดสอบ library =====
import sys
sys.path.insert(0, "/tmp")

import texttools

sample = """
  Python is a high-level, general-purpose programming language.   
  Its design philosophy emphasizes code readability.  
  Python was created by Guido van Rossum.  
"""

# Clean
cleaned = texttools.clean(sample)
print(f"Cleaned:\n{cleaned}\n")

# Analyze
print(f"Word count: {texttools.word_count(cleaned)}")
print(f"Char count: {texttools.char_count(cleaned)}")
print(f"Sentences: {texttools.char_count(cleaned, include_spaces=False)}")
print(f"Reading time: {texttools.reading_time(cleaned):.2f} min")

freq = texttools.word_frequency(cleaned, top_n=5)
print(f"Top words: {freq}")

# Format
wrapped = texttools.wrap(cleaned, width=50)
print(f"\nWrapped (50):\n{wrapped}")

truncated = texttools.truncate(cleaned, 100)
print(f"\nTruncated: {truncated}")

print(f"\nVersion: {texttools.__version__}")
```

## 11. สรุป Part 014

ใน Part นี้คุณได้เรียนรู้:

✅ **import** - `import module`, `from module import x`, `import as`, `from import *`
✅ **Standard Library** - `math`, `os`, `sys`, `datetime`, `random`, `collections`, `itertools`
✅ **สร้าง Module** - ไฟล์ `.py` ที่ใช้ import ได้
✅ **`__name__ == "__main__"`** - ป้องกัน code ทำงานเมื่อถูก import
✅ **Packages** - folder + `__init__.py`, relative imports, `__all__`
✅ **sys.path** - ค้นหา modules, เพิ่ม paths
✅ **pip** - install, uninstall, list, show, freeze
✅ **Virtual Environments** - `venv`, activate, deactivate
✅ **requirements.txt** - สร้าง, ติดตั้ง, แยก environments

---

## ➡️ ถัดไป: Part 015 - OOP Basics

*Part 014/100+ | Python Course - Beginner to World-Class*
