# Part 006: ฟังก์ชัน (Functions)
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- สร้างและเรียกใช้ฟังก์ชันได้
- เข้าใจ Parameters ทุกรูปแบบ (*args, **kwargs)
- ใช้ Return values ได้อย่างถูกต้อง
- เข้าใจ Scope (LEGB Rule)
- เขียน Docstrings ที่ดี
- ใช้ Lambda และ Higher-order functions

---

## 1. ฟังก์ชันพื้นฐาน

```python
# def - keyword สำหรับสร้างฟังก์ชัน
def greet():
    """ฟังก์ชันทักทายง่ายๆ"""
    print("สวัสดีครับ!")

# เรียกใช้ฟังก์ชัน
greet()     # สวัสดีครับ!
greet()     # เรียกกี่ครั้งก็ได้

# ฟังก์ชันกับ parameters
def greet_person(name):
    print(f"สวัสดี, {name}!")

greet_person("Alice")   # สวัสดี, Alice!
greet_person("Bob")     # สวัสดี, Bob!

# ฟังก์ชันกับ return value
def add(a, b):
    return a + b

result = add(3, 5)
print(result)   # 8
print(add(10, 20))  # 30
```

### 1.1 Docstrings

```python
def calculate_area(length, width):
    """
    คำนวณพื้นที่สี่เหลี่ยมผืนผ้า
    
    Args:
        length (float): ความยาว
        width (float): ความกว้าง
    
    Returns:
        float: พื้นที่
    
    Raises:
        ValueError: ถ้า length หรือ width เป็นค่าลบ
    
    Examples:
        >>> calculate_area(5, 3)
        15.0
        >>> calculate_area(10, 4)
        40.0
    """
    if length < 0 or width < 0:
        raise ValueError("ขนาดต้องไม่ติดลบ")
    return float(length * width)

# ดู docstring
print(calculate_area.__doc__)
help(calculate_area)

# ทดสอบ
print(calculate_area(5, 3))     # 15.0
print(calculate_area(10, 4))    # 40.0
```

---

## 2. Parameters และ Arguments

### 2.1 Positional Arguments

```python
def describe_person(name, age, city):
    print(f"{name}, อายุ {age}, จาก {city}")

# Positional - ต้องเรียงตามลำดับ
describe_person("Alice", 25, "Bangkok")
describe_person("Bob", 30, "Chiang Mai")

# ❌ ผิดลำดับ
# describe_person(25, "Alice", "Bangkok")  # ผิด!
```

### 2.2 Keyword Arguments

```python
def describe_person(name, age, city):
    print(f"{name}, อายุ {age}, จาก {city}")

# Keyword - ระบุชื่อ parameter ได้
describe_person(name="Alice", age=25, city="Bangkok")
describe_person(age=25, city="Bangkok", name="Alice")  # ลำดับไม่สำคัญ
describe_person("Alice", city="Bangkok", age=25)  # ผสม positional + keyword
```

### 2.3 Default Parameters

```python
def greet(name, greeting="สวัสดี"):
    print(f"{greeting}, {name}!")

greet("Alice")              # สวัสดี, Alice!
greet("Bob", "Hello")       # Hello, Bob!
greet("Charlie", greeting="ดีครับ")  # ดีครับ, Charlie!

# Default parameter ต้องมาหลัง non-default
def create_user(name, age, role="user", is_active=True):
    return {
        "name": name,
        "age": age,
        "role": role,
        "is_active": is_active
    }

user1 = create_user("Alice", 25)
print(user1)  # {'name': 'Alice', 'age': 25, 'role': 'user', 'is_active': True}

user2 = create_user("Bob", 30, role="admin")
print(user2)

# ⚠️ อย่าใช้ mutable เป็น default parameter!
# ❌ ผิด
def add_item_bad(item, items=[]):
    items.append(item)
    return items

print(add_item_bad("a"))  # ['a']
print(add_item_bad("b"))  # ['a', 'b']  (ไม่ใช่ ['b']!)
print(add_item_bad("c"))  # ['a', 'b', 'c']  (ไม่ถูกต้อง!)

# ✅ ถูกต้อง
def add_item_good(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items

print(add_item_good("a"))  # ['a']
print(add_item_good("b"))  # ['b']  (ถูกต้อง)
print(add_item_good("c"))  # ['c']  (ถูกต้อง)
```

### 2.4 *args - Arbitrary Positional Arguments

```python
# *args รับ arguments ไม่จำกัดจำนวน
def sum_all(*numbers):
    print(f"type: {type(numbers)}")  # <class 'tuple'>
    return sum(numbers)

print(sum_all(1, 2))           # 3
print(sum_all(1, 2, 3, 4, 5))  # 15
print(sum_all())               # 0

def greet_all(*names, greeting="สวัสดี"):
    for name in names:
        print(f"{greeting}, {name}!")

greet_all("Alice", "Bob", "Charlie")
greet_all("Dave", "Eve", greeting="Hello")

# Unpack list/tuple ด้วย *
numbers = [1, 2, 3, 4, 5]
print(sum_all(*numbers))    # เหมือนกับ sum_all(1, 2, 3, 4, 5)

def add(a, b, c):
    return a + b + c

args = [1, 2, 3]
print(add(*args))  # 6
```

### 2.5 **kwargs - Arbitrary Keyword Arguments

```python
# **kwargs รับ keyword arguments ไม่จำกัด
def print_info(**info):
    print(f"type: {type(info)}")  # <class 'dict'>
    for key, value in info.items():
        print(f"  {key}: {value}")

print_info(name="Alice", age=25, city="Bangkok")
print_info(language="Python", version="3.11", level="Intermediate")

# ผสม *args และ **kwargs
def create_profile(user_id, *tags, **metadata):
    return {
        "id": user_id,
        "tags": tags,
        "metadata": metadata
    }

profile = create_profile(
    123,
    "python", "developer", "backend",  # *tags
    name="Alice",                       # **metadata
    email="alice@example.com",
    active=True
)
print(profile)

# Unpack dict ด้วย **
kwargs = {"name": "Bob", "age": 30}
print_info(**kwargs)  # เหมือนกับ print_info(name="Bob", age=30)

def connect(host, port, timeout=30):
    print(f"Connecting to {host}:{port} (timeout={timeout}s)")

conn_params = {"host": "localhost", "port": 5432, "timeout": 60}
connect(**conn_params)
```

### 2.6 Keyword-Only Arguments

```python
# Parameters หลัง * ต้องระบุชื่อเสมอ
def send_email(to, subject, *, cc=None, bcc=None, html=False):
    print(f"To: {to}")
    print(f"Subject: {subject}")
    if cc:
        print(f"CC: {cc}")
    if bcc:
        print(f"BCC: {bcc}")
    print(f"Format: {'HTML' if html else 'Text'}")

send_email("alice@example.com", "Hello")
send_email("bob@example.com", "Meeting", cc="charlie@example.com", html=True)
# send_email("eve@example.com", "Test", "cc@example.com")  # TypeError!

# Positional-Only Arguments (Python 3.8+)
def power(base, exp, /, mod=None):
    if mod:
        return pow(base, exp, mod)
    return base ** exp

power(2, 10)           # ✅
power(2, exp=10)       # ❌ TypeError (base ต้อง positional)
power(2, 10, mod=1000) # ✅
```

---

## 3. Return Values

```python
# return หลายค่า
def min_max(numbers):
    return min(numbers), max(numbers)  # คืน tuple

result = min_max([3, 1, 4, 1, 5, 9, 2, 6])
print(result)       # (1, 9)
print(type(result)) # <class 'tuple'>

# Unpack return
minimum, maximum = min_max([3, 1, 4, 1, 5, 9, 2, 6])
print(f"Min: {minimum}, Max: {maximum}")

# Return None
def say_hello():
    print("Hello!")
    # ไม่มี return → คืน None

result = say_hello()
print(result)    # None

# Return ก่อนกำหนด (Early Return)
def find_first_even(numbers):
    for n in numbers:
        if n % 2 == 0:
            return n  # return ทันทีเมื่อพบ
    return None  # ไม่พบ

result = find_first_even([1, 3, 5, 4, 6])
print(result)   # 4

# Return dict สำหรับข้อมูลหลายค่า
def analyze_text(text):
    words = text.split()
    return {
        "word_count": len(words),
        "char_count": len(text),
        "unique_words": len(set(words)),
        "avg_word_length": sum(len(w) for w in words) / len(words) if words else 0,
    }

stats = analyze_text("Python is great Python is fun")
for key, value in stats.items():
    print(f"{key}: {value}")
```

---

## 4. Scope (LEGB Rule)

Python ค้นหาตัวแปรตาม LEGB:
- **L**ocal - ภายใน function ปัจจุบัน
- **E**nclosing - function ที่ห่อหุ้มอยู่
- **G**lobal - ระดับ module
- **B**uilt-in - Python built-ins (print, len, etc.)

```python
# Global scope
x = "global"

def outer():
    # Enclosing scope
    x = "enclosing"
    
    def inner():
        # Local scope
        x = "local"
        print(f"inner: {x}")   # local
    
    inner()
    print(f"outer: {x}")       # enclosing

outer()
print(f"global: {x}")          # global

# global keyword
counter = 0

def increment():
    global counter
    counter += 1

increment()
increment()
print(counter)  # 2

# nonlocal keyword (สำหรับ enclosing scope)
def make_counter():
    count = 0
    
    def increment():
        nonlocal count
        count += 1
        return count
    
    def get_count():
        return count
    
    return increment, get_count

inc, get = make_counter()
print(inc())    # 1
print(inc())    # 2
print(inc())    # 3
print(get())    # 3
```

### 4.1 Closure

```python
def make_multiplier(n):
    """Closure - ฟังก์ชันที่จำ environment ของ enclosing scope"""
    def multiplier(x):
        return x * n  # จำ n จาก enclosing scope
    return multiplier

double = make_multiplier(2)
triple = make_multiplier(3)

print(double(5))   # 10
print(triple(5))   # 15
print(double(10))  # 20

# ตัวอย่าง: Discount calculator
def make_discount_calculator(discount_percent):
    rate = 1 - (discount_percent / 100)
    
    def calculate(price):
        discounted = price * rate
        saved = price - discounted
        return discounted, saved
    
    return calculate

student_discount = make_discount_calculator(20)
vip_discount = make_discount_calculator(50)

original_price = 1000
discounted, saved = student_discount(original_price)
print(f"Student: {discounted:.2f} (ลด {saved:.2f})")

discounted, saved = vip_discount(original_price)
print(f"VIP: {discounted:.2f} (ลด {saved:.2f})")
```

---

## 5. Lambda Functions

```python
# Lambda - anonymous function ในบรรทัดเดียว
# รูปแบบ: lambda parameters: expression

# ฟังก์ชันปกติ
def square(x):
    return x ** 2

# Lambda แบบเดียวกัน
square_lambda = lambda x: x ** 2

print(square(5))          # 25
print(square_lambda(5))   # 25

# Lambda สำหรับ sorting
students = [
    {"name": "Charlie", "score": 85},
    {"name": "Alice", "score": 92},
    {"name": "Bob", "score": 78},
]

# เรียงตาม score
sorted_students = sorted(students, key=lambda s: s["score"])
for s in sorted_students:
    print(f"{s['name']}: {s['score']}")

# เรียงตาม score ลดลง
sorted_desc = sorted(students, key=lambda s: s["score"], reverse=True)

# เรียงตาม name
sorted_by_name = sorted(students, key=lambda s: s["name"])

# Lambda กับ filter, map
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# filter
evens = list(filter(lambda x: x % 2 == 0, numbers))
print(evens)  # [2, 4, 6, 8, 10]

# map
squares = list(map(lambda x: x ** 2, numbers))
print(squares)  # [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]

# reduce
from functools import reduce
product = reduce(lambda x, y: x * y, [1, 2, 3, 4, 5])
print(product)  # 120

# Lambda ใน dict
operations = {
    "add": lambda x, y: x + y,
    "sub": lambda x, y: x - y,
    "mul": lambda x, y: x * y,
    "div": lambda x, y: x / y if y != 0 else None,
}

print(operations["add"](5, 3))   # 8
print(operations["mul"](4, 7))   # 28
```

---

## 6. Higher-Order Functions

```python
# Higher-Order Function = ฟังก์ชันที่รับหรือคืนฟังก์ชัน

# map() - apply function to each element
numbers = [1, 2, 3, 4, 5]
squares = list(map(lambda x: x**2, numbers))
print(squares)  # [1, 4, 9, 16, 25]

# ✅ แนะนำใช้ list comprehension แทน map ในหลายกรณี
squares = [x**2 for x in numbers]

# filter() - กรองตาม condition
evens = list(filter(lambda x: x % 2 == 0, numbers))
print(evens)    # [2, 4]

# ✅ แนะนำใช้ list comprehension
evens = [x for x in numbers if x % 2 == 0]

# sorted() กับ key function
words = ["banana", "apple", "cherry", "date"]

sorted_by_length = sorted(words, key=len)
print(sorted_by_length)   # ['date', 'apple', 'banana', 'cherry']

sorted_alphabetically = sorted(words)
print(sorted_alphabetically)  # ['apple', 'banana', 'cherry', 'date']

# ฟังก์ชันรับฟังก์ชันเป็น parameter
def apply_twice(func, value):
    return func(func(value))

def add_one(x):
    return x + 1

print(apply_twice(add_one, 5))  # 7 (5 → 6 → 7)
print(apply_twice(lambda x: x * 2, 3))  # 12 (3 → 6 → 12)

# ฟังก์ชันคืนฟังก์ชัน
def make_adder(n):
    return lambda x: x + n

add5 = make_adder(5)
add10 = make_adder(10)

print(add5(3))   # 8
print(add10(3))  # 13

# functools.partial - สร้างฟังก์ชันใหม่จาก partial application
from functools import partial

def power(base, exp):
    return base ** exp

square = partial(power, exp=2)
cube = partial(power, exp=3)

print(square(5))    # 25
print(cube(5))      # 125
print(list(map(square, range(1, 6))))  # [1, 4, 9, 16, 25]
```

---

## 7. Recursive Functions

```python
# Recursive = ฟังก์ชันที่เรียกตัวเอง

# Factorial
def factorial(n):
    """n! = n × (n-1) × ... × 2 × 1"""
    if n <= 1:          # Base case
        return 1
    return n * factorial(n - 1)  # Recursive case

print(factorial(5))   # 120
print(factorial(10))  # 3628800

# Fibonacci
def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

for i in range(10):
    print(fibonacci(i), end=" ")  # 0 1 1 2 3 5 8 13 21 34
print()

# ⚠️ Fibonacci recursion ช้ามากสำหรับ n ใหญ่ ใช้ memoization แทน
from functools import lru_cache

@lru_cache(maxsize=None)
def fibonacci_fast(n):
    if n <= 1:
        return n
    return fibonacci_fast(n - 1) + fibonacci_fast(n - 2)

print(fibonacci_fast(50))  # เร็วมาก

# Binary Search (Recursive)
def binary_search(arr, target, left=0, right=None):
    if right is None:
        right = len(arr) - 1
    
    if left > right:
        return -1  # ไม่พบ
    
    mid = (left + right) // 2
    
    if arr[mid] == target:
        return mid
    elif arr[mid] < target:
        return binary_search(arr, target, mid + 1, right)
    else:
        return binary_search(arr, target, left, mid - 1)

sorted_array = [1, 3, 5, 7, 9, 11, 13, 15]
print(binary_search(sorted_array, 7))   # 3 (index)
print(binary_search(sorted_array, 10))  # -1 (ไม่พบ)

# Tree traversal
class TreeNode:
    def __init__(self, value, left=None, right=None):
        self.value = value
        self.left = left
        self.right = right

def inorder_traversal(node):
    """Left → Root → Right"""
    if node is None:
        return []
    return (inorder_traversal(node.left) + 
            [node.value] + 
            inorder_traversal(node.right))

# สร้าง tree
root = TreeNode(4,
    TreeNode(2, TreeNode(1), TreeNode(3)),
    TreeNode(6, TreeNode(5), TreeNode(7))
)

print(inorder_traversal(root))  # [1, 2, 3, 4, 5, 6, 7]
```

---

## 8. Function Annotations (Type Hints)

```python
from typing import List, Dict, Optional, Tuple, Union

# Type hints ช่วยให้โค้ดอ่านง่าย
def greet(name: str) -> str:
    return f"Hello, {name}!"

def add(a: int, b: int) -> int:
    return a + b

def get_stats(numbers: List[float]) -> Dict[str, float]:
    return {
        "mean": sum(numbers) / len(numbers),
        "min": min(numbers),
        "max": max(numbers),
    }

# Optional - อาจเป็น None
def find_user(user_id: int) -> Optional[Dict]:
    users = {1: {"name": "Alice"}, 2: {"name": "Bob"}}
    return users.get(user_id)

# Union - เป็นได้หลายชนิด
def process(data: Union[str, int, float]) -> str:
    return str(data)

# Python 3.10+ - ใช้ | แทน Union
def process_new(data: str | int | float) -> str:
    return str(data)

# Tuple return
def min_max(lst: List[int]) -> Tuple[int, int]:
    return min(lst), max(lst)

# ดู annotations
print(greet.__annotations__)
print(add.__annotations__)
```

---

## 9. ตัวอย่างโปรแกรม: Text Processing Library

```python
# text_utils.py
"""
ไลบรารีสำหรับ process text
"""

def word_count(text: str) -> int:
    """นับจำนวนคำในข้อความ"""
    return len(text.split())

def char_count(text: str, include_spaces: bool = True) -> int:
    """นับจำนวนตัวอักษร"""
    if include_spaces:
        return len(text)
    return len(text.replace(" ", ""))

def truncate(text: str, max_length: int, suffix: str = "...") -> str:
    """ตัดข้อความให้สั้นลง"""
    if len(text) <= max_length:
        return text
    return text[:max_length - len(suffix)] + suffix

def capitalize_words(text: str) -> str:
    """แปลงตัวแรกของทุกคำเป็นตัวใหญ่"""
    return " ".join(word.capitalize() for word in text.split())

def is_palindrome(text: str) -> bool:
    """ตรวจสอบว่าเป็น palindrome หรือไม่"""
    clean = "".join(c.lower() for c in text if c.isalnum())
    return clean == clean[::-1]

def extract_emails(text: str) -> list:
    """ดึง email addresses จากข้อความ"""
    import re
    pattern = r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b'
    return re.findall(pattern, text)

def slug_from_title(title: str) -> str:
    """แปลง title เป็น URL slug"""
    return "-".join(title.lower().split())

def repeat_string(text: str, times: int, separator: str = "") -> str:
    """ทำซ้ำ string"""
    return separator.join([text] * times)

# ทดสอบ
print("=== Text Utils Demo ===\n")

sample = "  Python is awesome!  "
print(f"Word count: {word_count(sample.strip())}")
print(f"Char count: {char_count(sample.strip())}")
print(f"Char count (no spaces): {char_count(sample.strip(), False)}")
print(f"Truncated: {truncate(sample.strip(), 15)}")
print(f"Capitalized: {capitalize_words('hello world from python')}")

palindromes = ["racecar", "A man a plan a canal Panama", "hello"]
for text in palindromes:
    result = "✅" if is_palindrome(text) else "❌"
    print(f"Palindrome '{text}': {result}")

email_text = "Contact us at info@example.com or support@example.org"
print(f"Emails found: {extract_emails(email_text)}")
print(f"Slug: {slug_from_title('Hello World From Python')}")
print(f"Repeat: {repeat_string('Ha', 3, ' ')}")
```

---

## 10. Exercises

### Exercise 1: Calculator Functions
```python
def add(a, b):
    return a + b

def subtract(a, b):
    return a - b

def multiply(a, b):
    return a * b

def divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b

def power(base, exp=2):
    return base ** exp

def calculator(operation, *args):
    operations = {
        'add': add,
        'subtract': subtract,
        'multiply': multiply,
        'divide': divide,
        'power': power,
    }
    
    if operation not in operations:
        raise ValueError(f"Unknown operation: {operation}")
    
    return operations[operation](*args)

# ทดสอบ
print(calculator('add', 5, 3))        # 8
print(calculator('multiply', 4, 6))   # 24
print(calculator('power', 3))         # 9
print(calculator('power', 2, 8))      # 256
```

### Exercise 2: Memoization
```python
def memoize(func):
    """Decorator-style memoization"""
    cache = {}
    
    def wrapper(*args):
        if args not in cache:
            cache[args] = func(*args)
        return cache[args]
    
    return wrapper

@memoize  # ใช้ @ syntax (Decorator - จะเรียนใน Part 018)
def expensive_calculation(n):
    """จำลองการคำนวณที่ใช้เวลานาน"""
    import time
    time.sleep(0.1)  # จำลอง delay
    return sum(range(n))

import time

# ครั้งแรก - ช้า
start = time.time()
result = expensive_calculation(100)
print(f"ครั้งแรก: {result} ({time.time()-start:.3f}s)")

# ครั้งที่สอง - เร็ว (จาก cache)
start = time.time()
result = expensive_calculation(100)
print(f"ครั้งที่สอง: {result} ({time.time()-start:.3f}s)")
```

---

## 11. สรุป Part 006

### สิ่งที่เรียนรู้:

✅ **def** - สร้างฟังก์ชัน  
✅ **Parameters** - positional, keyword, default  
✅ ***args** - รับ positional arguments ไม่จำกัด  
✅ ****kwargs** - รับ keyword arguments ไม่จำกัด  
✅ **return** - คืนค่า (single/multiple)  
✅ **Scope (LEGB)** - local, enclosing, global, built-in  
✅ **Closure** - ฟังก์ชันจำ enclosing scope  
✅ **Lambda** - anonymous function  
✅ **Higher-order functions** - map, filter, reduce  
✅ **Recursion** - ฟังก์ชันเรียกตัวเอง  
✅ **Type Hints** - annotations สำหรับ documentation  

### Quick Reference:

```python
# Basic function
def func(x, y=0):
    return x + y

# *args and **kwargs
def all_args(*args, **kwargs):
    pass  # args=tuple, kwargs=dict

# Lambda
double = lambda x: x * 2

# Scope
x = "global"
def func():
    global x    # แก้ global
    nonlocal y  # แก้ enclosing

# Type hints
def greet(name: str) -> str:
    return f"Hello, {name}!"
```

---

## ➡️ ถัดไป: Part 007 - Lists และ List Comprehension

ใน Part ถัดไป เราจะเรียนรู้:
- List operations ทั้งหมด
- List methods
- Slicing ขั้นสูง
- Sorting และ Searching
- List Comprehension ขั้นสูง

---

*Part 006/100+ | Python Course - Beginner to World-Class*
