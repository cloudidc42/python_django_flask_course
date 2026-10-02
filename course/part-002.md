# Part 002: ตัวแปรและชนิดข้อมูลพื้นฐาน
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจตัวแปร (Variables) และการตั้งชื่อ
- รู้จักชนิดข้อมูลทุกประเภทใน Python
- สามารถแปลงชนิดข้อมูลได้ (Type Conversion)
- รับ Input จากผู้ใช้ได้
- เข้าใจ None และ Boolean

---

## 1. ตัวแปร (Variables)

ตัวแปรคือ "กล่อง" สำหรับเก็บข้อมูล

### 1.1 การสร้างตัวแปร

```python
# Python ไม่ต้องประกาศชนิดข้อมูล (Dynamic Typing)
name = "Alice"           # str
age = 25                 # int
height = 1.75            # float
is_student = True        # bool
score = None             # NoneType

# Python รู้ชนิดข้อมูลเองอัตโนมัติ
print(type(name))        # <class 'str'>
print(type(age))         # <class 'int'>
print(type(height))      # <class 'float'>
print(type(is_student))  # <class 'bool'>
print(type(score))       # <class 'NoneType'>
```

### 1.2 กฎการตั้งชื่อตัวแปร

```python
# ✅ ถูกต้อง
name = "Alice"
user_name = "Bob"
_private = "secret"
__very_private = "more secret"
userName2 = "Charlie"        # camelCase (ไม่แนะนำแต่ใช้ได้)
name123 = "Dave"
CONSTANT_VALUE = 100         # constants ใช้ UPPER_CASE

# ❌ ผิด - ขึ้นต้นด้วยตัวเลข
# 1name = "Alice"           # SyntaxError

# ❌ ผิด - มีช่องว่าง
# my name = "Alice"         # SyntaxError

# ❌ ผิด - ใช้ special characters
# my-name = "Alice"         # SyntaxError
# my@name = "Alice"         # SyntaxError

# ❌ ผิด - ใช้ reserved keywords
# if = "value"              # SyntaxError
# for = 10                  # SyntaxError
# class = "MyClass"         # SyntaxError
```

### 1.3 Reserved Keywords ของ Python

```python
# ดู keywords ทั้งหมด
import keyword
print(keyword.kwlist)

# Output:
# ['False', 'None', 'True', 'and', 'as', 'assert', 'async', 'await',
#  'break', 'class', 'continue', 'def', 'del', 'elif', 'else', 'except',
#  'finally', 'for', 'from', 'global', 'if', 'import', 'in', 'is',
#  'lambda', 'nonlocal', 'not', 'or', 'pass', 'raise', 'return',
#  'try', 'while', 'with', 'yield']
```

### 1.4 การกำหนดค่าหลายตัวแปรพร้อมกัน

```python
# กำหนดค่าเดียวกันให้หลายตัวแปร
x = y = z = 0
print(x, y, z)  # 0 0 0

# กำหนดค่าหลายตัวพร้อมกัน (Multiple Assignment)
a, b, c = 1, 2, 3
print(a, b, c)  # 1 2 3

# Swap ค่าสองตัวแปร (Python วิธีสวยงาม)
x = 10
y = 20
x, y = y, x  # swap!
print(x, y)   # 20 10

# Unpack จาก list/tuple
first, second, third = [10, 20, 30]
print(first, second, third)  # 10 20 30

# Unpack แบบ *args
first, *rest = [1, 2, 3, 4, 5]
print(first)   # 1
print(rest)    # [2, 3, 4, 5]

*start, last = [1, 2, 3, 4, 5]
print(start)   # [1, 2, 3, 4]
print(last)    # 5

first, *middle, last = [1, 2, 3, 4, 5]
print(first)   # 1
print(middle)  # [2, 3, 4]
print(last)    # 5
```

### 1.5 Global vs Local Variables

```python
# Global variable
global_var = "ฉันอยู่ทุกที่"

def my_function():
    # Local variable
    local_var = "ฉันอยู่ในฟังก์ชัน"
    print(global_var)   # เข้าถึงได้
    print(local_var)    # เข้าถึงได้

my_function()
print(global_var)       # เข้าถึงได้
# print(local_var)      # NameError! local_var ไม่มีอยู่นอกฟังก์ชัน

# แก้ไข global variable ในฟังก์ชัน
counter = 0

def increment():
    global counter      # ประกาศว่าจะใช้ global variable
    counter += 1
    print(f"Counter: {counter}")

increment()  # Counter: 1
increment()  # Counter: 2
print(counter)  # 2
```

---

## 2. ชนิดข้อมูลพื้นฐาน (Basic Data Types)

Python มีชนิดข้อมูลหลัก 4 ประเภทพื้นฐาน:

```
int     - จำนวนเต็ม
float   - จำนวนทศนิยม
str     - ข้อความ
bool    - ค่าความจริง (True/False)
```

---

## 3. Integer (int) - จำนวนเต็ม

```python
# จำนวนเต็มพื้นฐาน
age = 25
year = 2024
temperature = -10
zero = 0

# Python ไม่จำกัดขนาด int!
big_number = 999999999999999999999999999
print(big_number)        # แสดงได้ปกติ

# ระบบเลขต่างๆ
binary = 0b1010          # เลขฐานสอง = 10
octal = 0o17             # เลขฐานแปด = 15
hexadecimal = 0xFF       # เลขฐานสิบหก = 255

print(binary)            # 10
print(octal)             # 15
print(hexadecimal)       # 255

# แปลงเป็นระบบอื่น
number = 255
print(bin(number))       # 0b11111111
print(oct(number))       # 0o377
print(hex(number))       # 0xff

# ตัวคั่นหลักพัน (Python 3.6+)
million = 1_000_000
billion = 1_000_000_000
print(million)           # 1000000
print(f"{million:,}")    # 1,000,000  (format)

# Operations
a = 17
b = 5
print(a + b)     # 22   บวก
print(a - b)     # 12   ลบ
print(a * b)     # 85   คูณ
print(a / b)     # 3.4  หาร (ได้ float เสมอ)
print(a // b)    # 3    หารเต็ม (integer division)
print(a % b)     # 2    เศษ (modulo)
print(a ** b)    # 1419857  ยกกำลัง

# abs() - ค่าสัมบูรณ์
print(abs(-10))  # 10
print(abs(10))   # 10

# divmod() - หารและเศษ
quotient, remainder = divmod(17, 5)
print(quotient, remainder)   # 3 2

# pow() - ยกกำลัง
print(pow(2, 10))    # 1024
print(2 ** 10)       # 1024 (เหมือนกัน)
print(pow(2, -1))    # 0.5  (ยกกำลังลบ)
```

---

## 4. Float (float) - จำนวนทศนิยม

```python
# Float พื้นฐาน
pi = 3.14159
price = 9.99
temperature = -3.5
small = 0.001

# Scientific notation
large = 1.5e10      # 1.5 × 10^10 = 15000000000.0
small_sci = 2.5e-4  # 2.5 × 10^-4 = 0.00025
print(large)        # 15000000000.0
print(small_sci)    # 0.00025

# Float precision ปัญหา!
print(0.1 + 0.2)         # 0.30000000000000004 (ไม่ใช่ 0.3!)
print(0.1 + 0.2 == 0.3)  # False

# แก้ด้วย round() หรือ decimal
print(round(0.1 + 0.2, 1))  # 0.3
print(round(0.1 + 0.2, 1) == 0.3)  # True

# ใช้ decimal module สำหรับความแม่นยำสูง
from decimal import Decimal
a = Decimal('0.1')
b = Decimal('0.2')
print(a + b)             # 0.3
print(a + b == Decimal('0.3'))  # True

# Float operations
x = 3.7
print(round(x))          # 4    (ปัดเศษ)
print(round(x, 1))       # 3.7  (ทศนิยม 1 ตำแหน่ง)
print(int(x))            # 3    (ตัดทศนิยม)
print(abs(-3.7))         # 3.7  (ค่าสัมบูรณ์)

# Special float values
import math
print(math.inf)          # inf  (อนันต์บวก)
print(-math.inf)         # -inf (อนันต์ลบ)
print(math.nan)          # nan  (Not a Number)
print(float('inf'))      # inf
print(float('nan'))      # nan

# ตรวจสอบ special values
print(math.isinf(math.inf))    # True
print(math.isnan(math.nan))    # True

# Float formatting
pi = 3.14159265358979
print(f"{pi:.2f}")       # 3.14   (ทศนิยม 2 ตำแหน่ง)
print(f"{pi:.4f}")       # 3.1416 (ทศนิยม 4 ตำแหน่ง)
print(f"{pi:10.2f}")     # "      3.14" (กว้าง 10 ตำแหน่ง)
print(f"{pi:010.2f}")    # "0000003.14" (เติม 0)
print(f"{pi:+.2f}")      # "+3.14" (แสดง sign)

# math module
import math
print(math.floor(3.7))   # 3  (ปัดลง)
print(math.ceil(3.2))    # 4  (ปัดขึ้น)
print(math.sqrt(16))     # 4.0 (รากที่สอง)
print(math.pi)           # 3.141592653589793
print(math.e)            # 2.718281828459045
```

---

## 5. String (str) - ข้อความ

```python
# การสร้าง string
name = "Alice"           # double quotes
greeting = 'Hello'       # single quotes
both = "It's a test"     # single quote ใน double quotes
escape = 'Say "Hello"'   # double quote ใน single quotes

# Multi-line string
poem = """
กลางคืน มาแล้ว
ดวงดาว ส่องฟ้า
Python เราเรียน
สนุกมาก
"""

# Raw string (ไม่แปล escape sequences)
path = r"C:\Users\Alice\Documents"
print(path)              # C:\Users\Alice\Documents

regex = r"\d+\.\d+"      # ใช้ใน regex
print(regex)             # \d+\.\d+

# String length
text = "Hello, World!"
print(len(text))         # 13

# String indexing (เริ่มที่ 0)
s = "Python"
print(s[0])              # P
print(s[1])              # y
print(s[-1])             # n (นับจากท้าย)
print(s[-2])             # o

# String slicing [start:end:step]
s = "Hello, World!"
print(s[0:5])            # Hello    (index 0-4)
print(s[7:])             # World!   (จาก index 7 ถึงจบ)
print(s[:5])             # Hello    (ตั้งแต่ต้นถึง index 4)
print(s[::2])            # Hlo ol!  (ทุกๆ 2 ตัว)
print(s[::-1])           # !dlroW ,olleH (กลับหลัง)

# String methods ที่ใช้บ่อย
text = "  Hello, World!  "

print(text.strip())              # "Hello, World!"   (ลบ whitespace หัวท้าย)
print(text.lstrip())             # "Hello, World!  " (ลบซ้าย)
print(text.rstrip())             # "  Hello, World!" (ลบขวา)

text = "Hello, World!"
print(text.upper())              # "HELLO, WORLD!"
print(text.lower())              # "hello, world!"
print(text.title())              # "Hello, World!" (ตัวใหญ่หน้าคำ)
print(text.swapcase())           # "hELLO, wORLD!"
print(text.capitalize())         # "Hello, world!"

print(text.replace("World", "Python"))  # "Hello, Python!"
print(text.replace("l", "L"))           # "HeLLo, WorLd!"
print(text.replace("l", "L", 1))        # "HeLlo, World!" (เปลี่ยนแค่ 1 ครั้ง)

print(text.count("l"))          # 3    (นับจำนวน)
print(text.find("World"))       # 7    (หา index ที่พบ, ถ้าไม่พบ -1)
print(text.index("World"))      # 7    (เหมือน find แต่ raise error ถ้าไม่พบ)
print(text.rfind("l"))          # 10   (หาจากท้าย)

print(text.startswith("Hello")) # True
print(text.endswith("!"))       # True
print(text.startswith("World")) # False

print(text.split(", "))         # ['Hello', 'World!']
print(text.split())             # ['Hello,', 'World!'] (split by whitespace)

words = ["Hello", "World", "Python"]
print(", ".join(words))         # "Hello, World, Python"
print(" ".join(words))          # "Hello World Python"
print("".join(words))           # "HelloWorldPython"

print(text.isalpha())           # False (มี , และ ! และ space)
print("Hello".isalpha())        # True  (มีแต่ตัวอักษร)
print("123".isdigit())          # True  (มีแต่ตัวเลข)
print("Hello123".isalnum())     # True  (ตัวอักษรหรือตัวเลข)
print("   ".isspace())          # True  (มีแต่ whitespace)

print("hello".isupper())        # False
print("HELLO".isupper())        # True
print("hello".islower())        # True

print(text.center(30, '*'))     # "********Hello, World!*********"
print(text.ljust(30, '-'))      # "Hello, World!-----------------"
print(text.rjust(30, '-'))      # "-----------------Hello, World!"
print("42".zfill(8))            # "00000042"

# String formatting
name = "Alice"
age = 25
score = 95.678

# f-string (Python 3.6+) - แนะนำ
print(f"Name: {name}, Age: {age}")
print(f"Score: {score:.2f}")         # Score: 95.68
print(f"Pi = {3.14159:.3f}")         # Pi = 3.142
print(f"{age:05d}")                  # 00025
print(f"{1000000:,}")                # 1,000,000
print(f"{0.75:.1%}")                 # 75.0%

# format() method
print("Name: {}, Age: {}".format(name, age))
print("Name: {0}, Age: {1}".format(name, age))
print("Name: {n}, Age: {a}".format(n=name, a=age))

# % formatting (แบบเก่า ยังใช้ได้)
print("Name: %s, Age: %d" % (name, age))
print("Score: %.2f" % score)

# Escape sequences
print("Line 1\nLine 2")     # newline
print("Tab\there")          # tab
print("Quote: \"hello\"")   # double quote
print("Backslash: \\")      # backslash
print("Unicode: \u0E44\u0E17\u0E22")  # ไทย

# String multiplication
print("-" * 50)             # --------------------------------------------------
print("Ha" * 3)             # HaHaHa

# String concatenation
first = "Hello"
second = " World"
combined = first + second    # "Hello World"
print(combined)
```

---

## 6. Boolean (bool) - ค่าความจริง

```python
# Boolean values
is_active = True
is_deleted = False

# Boolean operations
print(True and True)    # True
print(True and False)   # False
print(False and True)   # False
print(False and False)  # False

print(True or True)     # True
print(True or False)    # True
print(False or True)    # True
print(False or False)   # False

print(not True)         # False
print(not False)        # True

# Comparison operators → ได้ bool
x = 10
print(x > 5)            # True
print(x < 5)            # False
print(x >= 10)          # True
print(x <= 10)          # True
print(x == 10)          # True
print(x != 10)          # False

# bool() function
print(bool(1))          # True
print(bool(0))          # False
print(bool(-1))         # True  (ทุกจำนวนที่ไม่ใช่ 0)
print(bool(""))         # False (string ว่าง)
print(bool("Hello"))    # True
print(bool([]))         # False (list ว่าง)
print(bool([1, 2]))     # True
print(bool(None))       # False

# Truthy vs Falsy
# Falsy values:
# - False
# - None
# - 0, 0.0, 0j
# - "", '', ``
# - [], (), {}, set()
# - range(0)

# Truthy values:
# - ทุกอย่างที่ไม่ใช่ falsy

def check_truthy(value):
    if value:
        print(f"{repr(value)} is Truthy")
    else:
        print(f"{repr(value)} is Falsy")

check_truthy(True)      # True is Truthy
check_truthy(False)     # False is Falsy
check_truthy(1)         # 1 is Truthy
check_truthy(0)         # 0 is Falsy
check_truthy("Hello")   # 'Hello' is Truthy
check_truthy("")        # '' is Falsy
check_truthy([1, 2])    # [1, 2] is Truthy
check_truthy([])        # [] is Falsy
check_truthy(None)      # None is Falsy

# Short-circuit evaluation
def say_hello():
    print("Hello!")
    return True

# and - ถ้าตัวแรก False ไม่ประเมินตัวที่สอง
print(False and say_hello())  # False (ไม่พิมพ์ Hello!)
print(True and say_hello())   # Hello! True

# or - ถ้าตัวแรก True ไม่ประเมินตัวที่สอง
print(True or say_hello())    # True (ไม่พิมพ์ Hello!)
print(False or say_hello())   # Hello! True

# ใช้ประโยชน์จาก short-circuit
user = None
name = user or "Guest"       # ถ้า user เป็น None ใช้ "Guest"
print(name)  # Guest

user = "Alice"
name = user or "Guest"       # ถ้า user มีค่า ใช้ user
print(name)  # Alice

# Walrus operator := (Python 3.8+)
numbers = [1, 2, 3, 4, 5]
if (n := len(numbers)) > 3:
    print(f"List has {n} elements (more than 3)")
```

---

## 7. None - ค่าว่าง

```python
# None แทน "ไม่มีค่า"
result = None
name = None
data = None

# ตรวจสอบ None
if result is None:          # ✅ แนะนำ
    print("No result")

if result == None:          # ⚠️ ใช้ได้แต่ไม่แนะนำ
    print("No result")

# None ใน function ที่ไม่ return ค่า
def say_hi():
    print("Hi!")
    # ไม่มี return → คืนค่า None

result = say_hi()           # Hi!
print(result)               # None
print(type(result))         # <class 'NoneType'>

# ใช้ None เป็น default parameter
def greet(name=None):
    if name is None:
        return "Hello, Guest!"
    return f"Hello, {name}!"

print(greet())              # Hello, Guest!
print(greet("Alice"))       # Hello, Alice!

# None vs False
print(None == False)        # False (ต่างกัน)
print(None is False)        # False
print(bool(None))           # False (ทั้งคู่เป็น falsy แต่ไม่เท่ากัน)
```

---

## 8. Type Conversion (การแปลงชนิดข้อมูล)

### 8.1 Implicit Conversion (อัตโนมัติ)

```python
# Python แปลงอัตโนมัติในบางกรณี
a = 5        # int
b = 2.0      # float
result = a + b   # int + float = float
print(result)    # 7.0
print(type(result))  # <class 'float'>

# bool ถือว่าเป็น subclass ของ int
print(True + 1)     # 2
print(False + 1)    # 1
print(True + True)  # 2
print(True * 5)     # 5
```

### 8.2 Explicit Conversion (บังคับ)

```python
# int() - แปลงเป็น integer
print(int("123"))        # 123
print(int("  42  "))     # 42 (ลบ whitespace อัตโนมัติ)
print(int(3.7))          # 3  (ตัดทศนิยม ไม่ปัด)
print(int(True))         # 1
print(int(False))        # 0
# print(int("12.5"))     # ValueError! (ต้องแปลงผ่าน float ก่อน)
print(int(float("12.5"))) # 12

# แปลงจากฐานอื่น
print(int("FF", 16))     # 255  (hex → int)
print(int("1010", 2))    # 10   (binary → int)
print(int("17", 8))      # 15   (octal → int)

# float() - แปลงเป็น float
print(float("3.14"))     # 3.14
print(float("1e-3"))     # 0.001
print(float(5))          # 5.0
print(float(True))       # 1.0
print(float("inf"))      # inf
print(float("-inf"))     # -inf
# print(float("hello"))  # ValueError!

# str() - แปลงเป็น string
print(str(42))           # "42"
print(str(3.14))         # "3.14"
print(str(True))         # "True"
print(str(None))         # "None"
print(str([1, 2, 3]))    # "[1, 2, 3]"

# bool() - แปลงเป็น boolean
print(bool(1))           # True
print(bool(0))           # False
print(bool(""))          # False
print(bool("text"))      # True
print(bool([]))          # False
print(bool([0]))         # True (list ไม่ว่าง = True)
print(bool(None))        # False

# list() - แปลงเป็น list
print(list("Hello"))     # ['H', 'e', 'l', 'l', 'o']
print(list((1, 2, 3)))   # [1, 2, 3]
print(list({1, 2, 3}))   # [1, 2, 3] (ไม่รับประกันลำดับ)
print(list(range(5)))    # [0, 1, 2, 3, 4]

# tuple() - แปลงเป็น tuple
print(tuple([1, 2, 3]))  # (1, 2, 3)
print(tuple("Hello"))    # ('H', 'e', 'l', 'l', 'o')

# set() - แปลงเป็น set
print(set([1, 2, 2, 3])) # {1, 2, 3} (ไม่มีซ้ำ)
print(set("Hello"))      # {'H', 'e', 'l', 'o'} (ไม่มีซ้ำ, ไม่เรียงลำดับ)

# dict() - แปลงเป็น dict
print(dict([("a", 1), ("b", 2)]))  # {'a': 1, 'b': 2}
print(dict(a=1, b=2))              # {'a': 1, 'b': 2}
```

### 8.3 Safe Conversion ด้วย try/except

```python
def safe_int(value, default=0):
    """แปลงเป็น int อย่างปลอดภัย"""
    try:
        return int(value)
    except (ValueError, TypeError):
        return default

print(safe_int("123"))      # 123
print(safe_int("abc"))      # 0  (ใช้ default)
print(safe_int("abc", -1))  # -1 (ใช้ default ที่กำหนด)
print(safe_int(None))       # 0
print(safe_int(3.9))        # 3

def safe_float(value, default=0.0):
    """แปลงเป็น float อย่างปลอดภัย"""
    try:
        return float(value)
    except (ValueError, TypeError):
        return default

print(safe_float("3.14"))   # 3.14
print(safe_float("abc"))    # 0.0
```

---

## 9. Input จากผู้ใช้

```python
# input() - รับข้อมูลจาก keyboard
name = input("กรุณากรอกชื่อของคุณ: ")
print(f"สวัสดี, {name}!")

# input() คืนค่าเป็น string เสมอ!
age_str = input("กรอกอายุของคุณ: ")
print(type(age_str))      # <class 'str'>

# ต้องแปลงชนิดข้อมูลเอง
age = int(input("กรอกอายุ: "))
height = float(input("กรอกส่วนสูง (เมตร): "))

# Safe input
def get_int_input(prompt):
    while True:
        try:
            value = int(input(prompt))
            return value
        except ValueError:
            print("กรุณากรอกตัวเลขจำนวนเต็ม")

age = get_int_input("กรอกอายุ: ")
print(f"อายุของคุณคือ {age} ปี")

# รับค่าหลายค่าในบรรทัดเดียว
a, b = input("กรอก 2 ตัวเลข (คั่นด้วยช่องว่าง): ").split()
a, b = int(a), int(b)
print(f"{a} + {b} = {a + b}")

# หรือแบบสั้นกว่า
a, b = map(int, input("กรอก 2 ตัวเลข: ").split())
print(f"Sum: {a + b}")

# รับ list ของตัวเลข
numbers = list(map(int, input("กรอกตัวเลขหลายตัว: ").split()))
print(f"Sum: {sum(numbers)}")
print(f"Average: {sum(numbers) / len(numbers):.2f}")
```

---

## 10. การตรวจสอบชนิดข้อมูล

```python
# type() - ดูชนิดข้อมูล
print(type(42))             # <class 'int'>
print(type(3.14))           # <class 'float'>
print(type("Hello"))        # <class 'str'>
print(type(True))           # <class 'bool'>
print(type(None))           # <class 'NoneType'>
print(type([1, 2, 3]))      # <class 'list'>

# isinstance() - ตรวจสอบชนิดข้อมูล (แนะนำ)
x = 42
print(isinstance(x, int))       # True
print(isinstance(x, float))     # False
print(isinstance(x, (int, float)))  # True (เป็น int หรือ float)

name = "Alice"
print(isinstance(name, str))    # True

# bool เป็น subclass ของ int
print(isinstance(True, bool))   # True
print(isinstance(True, int))    # True  (bool เป็น int ด้วย!)

# type() vs isinstance()
print(type(True) == int)        # False (True เป็น bool ไม่ใช่ int)
print(isinstance(True, int))    # True  (bool เป็น subclass ของ int)

# ตรวจสอบ callable
def my_func():
    pass

print(callable(my_func))        # True
print(callable(42))             # False
print(callable(print))          # True
```

---

## 11. Variables และ Memory (เบื้องต้น)

```python
# Python ใช้ references ไม่ใช่ copies สำหรับ mutable objects
a = [1, 2, 3]
b = a               # b ชี้ไปที่ list เดียวกัน!

b.append(4)
print(a)            # [1, 2, 3, 4]  (a เปลี่ยนด้วย!)
print(b)            # [1, 2, 3, 4]

# ตรวจสอบ identity (id)
print(id(a))        # memory address
print(id(b))        # address เดียวกัน
print(a is b)       # True (เป็น object เดียวกัน)

# ทำ copy จริงๆ
a = [1, 2, 3]
b = a.copy()        # หรือ b = list(a) หรือ b = a[:]

b.append(4)
print(a)            # [1, 2, 3]     (a ไม่เปลี่ยน)
print(b)            # [1, 2, 3, 4]
print(a is b)       # False

# Immutable objects (int, str, tuple, float, bool) ไม่มีปัญหานี้
x = 10
y = x
y = 20          # สร้าง int ใหม่ ไม่กระทบ x
print(x)        # 10
print(y)        # 20

# String interning (Python optimize ตัวเลขและ string สั้นๆ)
a = "hello"
b = "hello"
print(a is b)   # True (มักจะ True สำหรับ string สั้นๆ)

a = "hello world with spaces"
b = "hello world with spaces"
print(a is b)   # อาจเป็น True หรือ False (ขึ้นอยู่กับ implementation)
print(a == b)   # True เสมอ (ใช้ == สำหรับเปรียบเทียบ value)
```

---

## 12. ตัวอย่างโปรแกรม: ระบบข้อมูลนักเรียน

```python
# student_info.py
"""
ระบบข้อมูลนักเรียนพื้นฐาน
ใช้ชนิดข้อมูลพื้นฐานทั้งหมด
"""

# ข้อมูลนักเรียน
student_id = 12345                  # int
student_name = "สมชาย ใจดี"          # str
age = 18                            # int
gpa = 3.85                          # float
is_scholarship = True               # bool
advisor = None                      # NoneType (ยังไม่มีอาจารย์ที่ปรึกษา)
major = "Computer Science"          # str
year = 2                            # int

# แสดงข้อมูล
print("=" * 50)
print("  ระบบข้อมูลนักเรียน")
print("=" * 50)
print(f"รหัสนักศึกษา: {student_id:05d}")
print(f"ชื่อ-นามสกุล: {student_name}")
print(f"อายุ:         {age} ปี")
print(f"GPA:          {gpa:.2f}")
print(f"สาขาวิชา:     {major}")
print(f"ชั้นปี:       {year}")
print(f"ทุนการศึกษา:  {'ได้รับทุน' if is_scholarship else 'ไม่ได้รับทุน'}")
print(f"อาจารย์ที่ปรึกษา: {advisor if advisor is not None else 'ยังไม่กำหนด'}")
print("=" * 50)

# คำนวณข้อมูลเพิ่มเติม
credit_hours = 18
total_credits = year * 36
remaining_credits = 128 - total_credits  # ต้องเรียน 128 หน่วยกิต

print(f"\nข้อมูลการเรียน:")
print(f"หน่วยกิตปัจจุบัน: {credit_hours}")
print(f"หน่วยกิตสะสม:    {total_credits}")
print(f"หน่วยกิตที่เหลือ: {remaining_credits}")
print(f"เปอร์เซ็นต์ความก้าวหน้า: {(total_credits / 128 * 100):.1f}%")

# ตรวจสอบสถานะ
excellent = gpa >= 3.5
good = gpa >= 3.0 and gpa < 3.5
average = gpa >= 2.0 and gpa < 3.0
failing = gpa < 2.0

if excellent:
    status = "เกียรตินิยม"
elif good:
    status = "ดีมาก"
elif average:
    status = "พอใช้"
else:
    status = "ต้องปรับปรุง"

print(f"\nสถานะทางการเรียน: {status}")
```

---

## 13. Exercises (แบบฝึกหัด)

### Exercise 1: ตัวแปรและชนิดข้อมูล
```python
# สร้างตัวแปรสำหรับข้อมูลส่วนตัวของคุณ:
# - ชื่อ (string)
# - อายุ (int)
# - ส่วนสูง (float)
# - เป็นนักเรียนหรือไม่ (bool)
# - งานอดิเรก (list)
# - ที่อยู่ (ยังไม่มี = None)
# แล้วแสดงข้อมูลทั้งหมดพร้อมชนิดข้อมูล

name = "สมชาย"
age = 22
height = 1.75
is_student = True
hobbies = ["อ่านหนังสือ", "เล่นเกม", "โปรแกรม"]
address = None

print(f"ชื่อ: {name} (type: {type(name).__name__})")
print(f"อายุ: {age} (type: {type(age).__name__})")
print(f"ส่วนสูง: {height} (type: {type(height).__name__})")
print(f"เป็นนักเรียน: {is_student} (type: {type(is_student).__name__})")
print(f"งานอดิเรก: {hobbies} (type: {type(hobbies).__name__})")
print(f"ที่อยู่: {address} (type: {type(address).__name__})")
```

### Exercise 2: Type Conversion
```python
# แปลงชนิดข้อมูลและแสดงผล
data = ["123", "45.6", "True", "0", "", "100"]

for item in data:
    print(f"'{item}':")
    print(f"  int: {int(float(item)) if item else 0}")
    print(f"  float: {float(item) if item else 0.0}")
    print(f"  bool: {bool(item)}")
    print()
```

### Exercise 3: โปรแกรมรับข้อมูล
```python
# สร้างโปรแกรมที่รับข้อมูลจาก user แล้วแสดงผล

print("=== กรอกข้อมูลส่วนตัว ===")
name = input("ชื่อ: ")
age = int(input("อายุ: "))
height = float(input("ส่วนสูง (ซม.): "))

# คำนวณ
height_m = height / 100
bmi = 70 / (height_m ** 2)  # สมมติน้ำหนัก 70 กก.
birth_year = 2024 - age

print(f"\n=== ข้อมูลของคุณ ===")
print(f"ชื่อ: {name}")
print(f"อายุ: {age} ปี")
print(f"เกิดปี: {birth_year}")
print(f"ส่วนสูง: {height} ซม. ({height_m:.2f} ม.)")
print(f"BMI: {bmi:.1f}")
```

---

## 14. สรุป Part 002

### สิ่งที่เรียนรู้:

✅ **ตัวแปร** - กฎการตั้งชื่อ, กำหนดค่า, scope  
✅ **int** - จำนวนเต็ม, operations, ระบบเลขต่างๆ  
✅ **float** - ทศนิยม, precision, math operations  
✅ **str** - string methods, formatting, slicing  
✅ **bool** - True/False, Truthy/Falsy, short-circuit  
✅ **None** - ค่าว่าง, ตรวจสอบ  
✅ **Type Conversion** - int(), float(), str(), bool()  
✅ **input()** - รับข้อมูลจาก user  

### Quick Reference:

```python
# สร้างตัวแปร
x = 10                  # int
y = 3.14               # float  
s = "Hello"            # str
b = True               # bool
n = None               # NoneType

# ตรวจสอบชนิด
type(x)                # <class 'int'>
isinstance(x, int)     # True

# แปลงชนิด
int("123")             # 123
float("3.14")          # 3.14
str(42)                # "42"
bool(0)                # False

# รับ input
x = int(input("กรอกตัวเลข: "))
```

---

## ➡️ ถัดไป: Part 003 - ตัวดำเนินการและนิพจน์

ใน Part ถัดไป เราจะเรียนรู้:
- Arithmetic Operators (+, -, *, /, //, %, **)
- Comparison Operators (==, !=, >, <, >=, <=)
- Logical Operators (and, or, not)
- Assignment Operators (=, +=, -=, *=)
- Bitwise Operators
- Operator Precedence

---

*Part 002/100+ | Python Course - Beginner to World-Class*
