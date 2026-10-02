# Part 020: Functional Programming (การเขียนโปรแกรมเชิงฟังก์ชัน)
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้
- เข้าใจแนวคิด Functional Programming
- ใช้ `map()`, `filter()`, `reduce()` อย่างคล่องแคล่ว
- เขียน Lambda Functions ได้อย่างเหมาะสม
- ใช้ `zip()`, `enumerate()`, `sorted()` with key functions
- ใช้ `all()`, `any()` สำหรับ boolean operations
- ใช้ `functools` module: `partial`, `reduce`, `lru_cache`

---

## 1. แนวคิด Functional Programming

Functional Programming (FP) คือแนวคิดการเขียนโปรแกรมที่:
- ใช้ **Pure Functions** - output ขึ้นอยู่กับ input เท่านั้น ไม่มี side effects
- **Immutable Data** - ไม่แก้ไขข้อมูลต้นฉบับ สร้างใหม่แทน
- **Higher-Order Functions** - ฟังก์ชันรับหรือคืนฟังก์ชัน

```python
# Pure Function - ผลลัพธ์เหมือนเดิมทุกครั้ง
def add(a, b):
    return a + b  # ไม่แก้ไขสิ่งภายนอก

print(add(2, 3))  # 5
print(add(2, 3))  # 5 เสมอ

# Impure Function - มี side effect
total = 0
def add_to_total(n):
    global total
    total += n  # แก้ไข global state (ไม่ดี)
    return total

# Higher-Order Function
def apply_twice(func, value):
    """รับ function และ value, apply function 2 ครั้ง"""
    return func(func(value))

def double(x):
    return x * 2

print(apply_twice(double, 3))   # 12 (3 -> 6 -> 12)
print(apply_twice(lambda x: x + 1, 5))  # 7

# Immutable approach
original = [1, 2, 3, 4, 5]
# ไม่ดี: แก้ไข original
# original.append(6)

# ดี: สร้างใหม่
new_list = original + [6]
print("original:", original)  # [1, 2, 3, 4, 5]
print("new_list:", new_list)  # [1, 2, 3, 4, 5, 6]
```

---

## 2. Lambda Functions

```python
# Lambda = anonymous function สำหรับ expressions ง่ายๆ
# syntax: lambda arguments: expression

# ฟังก์ชันปกติ
def square(x):
    return x ** 2

# Lambda เทียบเท่า
square_lambda = lambda x: x ** 2

print(square(5))        # 25
print(square_lambda(5)) # 25

# Lambda กับหลาย arguments
add = lambda a, b: a + b
print(add(3, 4))  # 7

power = lambda base, exp: base ** exp
print(power(2, 10))  # 1024

# Lambda กับ default values
greet = lambda name, greeting="สวัสดี": f"{greeting}, {name}!"
print(greet("สมชาย"))          # สวัสดี, สมชาย!
print(greet("อรอุมา", "ดีจ้า"))  # ดีจ้า, อรอุมา!

# Lambda ใน data structures
operations = {
    'add': lambda a, b: a + b,
    'sub': lambda a, b: a - b,
    'mul': lambda a, b: a * b,
    'div': lambda a, b: a / b if b != 0 else None
}

for op_name, op_func in operations.items():
    result = op_func(10, 3)
    print(f"{op_name}(10, 3) = {result}")

# เมื่อไหร่ NOT ควรใช้ Lambda
# ❌ ซับซ้อนเกินไป
# bad = lambda x: x if x > 0 else (-x if x < 0 else 0)

# ✅ ใช้ def แทน
def abs_value(x):
    if x > 0:
        return x
    elif x < 0:
        return -x
    return 0
```

---

## 3. map() - แปลงข้อมูล

```python
# map(function, iterable) - apply function กับทุก element
# คืน map object (iterator, lazy)

numbers = [1, 2, 3, 4, 5]

# วิธีปกติ
squares_loop = []
for n in numbers:
    squares_loop.append(n ** 2)

# ด้วย map()
squares_map = list(map(lambda x: x ** 2, numbers))

# ด้วย List Comprehension (แนะนำ Python 3)
squares_comp = [x ** 2 for x in numbers]

print(squares_loop)  # [1, 4, 9, 16, 25]
print(squares_map)   # [1, 4, 9, 16, 25]
print(squares_comp)  # [1, 4, 9, 16, 25]

# map() กับ function ที่กำหนดเอง
def celsius_to_fahrenheit(c):
    return (c * 9/5) + 32

temps_celsius = [0, 20, 37, 100]
temps_fahrenheit = list(map(celsius_to_fahrenheit, temps_celsius))
print(temps_fahrenheit)  # [32.0, 68.0, 98.6, 212.0]

# map() กับหลาย iterables
a = [1, 2, 3]
b = [10, 20, 30]
products = list(map(lambda x, y: x * y, a, b))
print(products)  # [10, 40, 90]

# ใช้งานจริง: แปลงข้อมูล users
users_raw = [
    "  สมชาย ใจดี  ",
    "อรอุมา  มีสุข",
    "  พีรพล  วิเศษ"
]
users_clean = list(map(str.strip, users_raw))
users_upper = list(map(str.upper, ["hello", "world", "python"]))
print(users_clean)
print(users_upper)  # ['HELLO', 'WORLD', 'PYTHON']

# map() กับ int() สำหรับแปลง types
numbers_str = ['1', '2', '3', '4', '5']
numbers_int = list(map(int, numbers_str))
print(numbers_int)        # [1, 2, 3, 4, 5]
print(sum(numbers_int))   # 15
```

---

## 4. filter() - กรองข้อมูล

```python
# filter(function, iterable) - เก็บ element ที่ function คืน True
# คืน filter object (iterator, lazy)

numbers = range(-5, 6)  # -5 ถึง 5

# กรองเลขบวก
positives = list(filter(lambda x: x > 0, numbers))
print("เลขบวก:", positives)  # [1, 2, 3, 4, 5]

# กรองเลขคี่
odds = list(filter(lambda x: x % 2 != 0, range(10)))
print("เลขคี่:", odds)  # [1, 3, 5, 7, 9]

# filter() กับ None (กรองค่า falsy)
mixed = [0, 1, '', 'hello', None, False, True, [], [1, 2]]
truthy = list(filter(None, mixed))
print("ค่า truthy:", truthy)  # [1, 'hello', True, [1, 2]]

# ตัวอย่างจริง: กรองข้อมูล
students = [
    {'name': 'อลิสา', 'grade': 85, 'passed': True},
    {'name': 'บอม', 'grade': 42, 'passed': False},
    {'name': 'ชาลี', 'grade': 91, 'passed': True},
    {'name': 'ดาว', 'grade': 55, 'passed': False},
    {'name': 'เอก', 'grade': 78, 'passed': True},
]

# กรองนักเรียนที่ผ่าน
passed = list(filter(lambda s: s['passed'], students))
print("นักเรียนที่ผ่าน:", [s['name'] for s in passed])
# ['อลิสา', 'ชาลี', 'เอก']

# กรองนักเรียนที่ได้เกรดสูงกว่า 80
high_achievers = list(filter(lambda s: s['grade'] > 80, students))
print("เกรดสูง:", [s['name'] for s in high_achievers])
# ['อลิสา', 'ชาลี']

# เทียบกับ List Comprehension
high_achievers_comp = [s for s in students if s['grade'] > 80]
print("เทียบเท่า:", [s['name'] for s in high_achievers_comp])

# กรองไฟล์ Python
files = ['app.py', 'data.csv', 'main.py', 'README.md', 'utils.py', 'config.json']
py_files = list(filter(lambda f: f.endswith('.py'), files))
print("Python files:", py_files)  # ['app.py', 'main.py', 'utils.py']
```

---

## 5. reduce() - รวมข้อมูล

```python
from functools import reduce

# reduce(function, iterable, initial) - รวม iterable เป็น value เดียว
# ทำงานแบบ: acc = initial; for x in iterable: acc = func(acc, x)

numbers = [1, 2, 3, 4, 5]

# Sum
total = reduce(lambda acc, x: acc + x, numbers)
print("sum:", total)  # 15

# Product
product = reduce(lambda acc, x: acc * x, numbers)
print("product:", product)  # 120

# Max
maximum = reduce(lambda a, b: a if a > b else b, numbers)
print("max:", maximum)  # 5

# กับ initial value
total_with_initial = reduce(lambda acc, x: acc + x, numbers, 100)
print("sum + initial 100:", total_with_initial)  # 115

# ตัวอย่างจริง
# Flatten nested list
nested = [[1, 2], [3, 4], [5, 6]]
flat = reduce(lambda acc, lst: acc + lst, nested)
print("flatten:", flat)  # [1, 2, 3, 4, 5, 6]

# Build string
words = ['Python', 'Django', 'Flask', 'FastAPI']
sentence = reduce(lambda s, w: s + ', ' + w, words)
print("joined:", sentence)  # Python, Django, Flask, FastAPI

# Compose functions
def compose(*functions):
    """สร้าง function ที่เป็น composition ของหลาย functions"""
    return reduce(lambda f, g: lambda x: f(g(x)), functions)

double = lambda x: x * 2
add_one = lambda x: x + 1
square = lambda x: x ** 2

# เทียบเท่า double(add_one(square(x)))
transform = compose(double, add_one, square)
print(transform(3))   # double(add_one(square(3))) = double(add_one(9)) = double(10) = 20

# เมื่อไหร่ใช้ reduce vs ทางอื่น
print(sum(numbers))        # ใช้ sum() แทน reduce สำหรับการบวก
print(max(numbers))        # ใช้ max() แทน reduce
print(','.join(words))     # ใช้ join() แทน reduce สำหรับ string
```

---

## 6. zip() - จับคู่ข้อมูล

```python
# zip(iter1, iter2, ...) - จับคู่ elements จากหลาย iterables
# หยุดเมื่อ iterable ที่สั้นที่สุดหมด

names = ['อลิสา', 'บอม', 'ชาลี']
ages = [25, 30, 28]
scores = [85, 92, 78]

# จับคู่ 2 lists
pairs = list(zip(names, ages))
print("pairs:", pairs)  # [('อลิสา', 25), ('บอม', 30), ('ชาลี', 28)]

# จับคู่ 3 lists
triples = list(zip(names, ages, scores))
print("triples:", triples)

# ใช้กับ dict
student_dict = dict(zip(names, scores))
print("dict:", student_dict)
# {'อลิสา': 85, 'บอม': 92, 'ชาลี': 78}

# Unzip (inverse of zip)
zipped = [(1, 'a'), (2, 'b'), (3, 'c')]
numbers, letters = zip(*zipped)  # unpack
print("numbers:", numbers)  # (1, 2, 3)
print("letters:", letters)  # ('a', 'b', 'c')

# zip_longest จาก itertools - หยุดเมื่อยาวที่สุดหมด
from itertools import zip_longest

list1 = [1, 2, 3, 4, 5]
list2 = ['a', 'b', 'c']

with_longest = list(zip_longest(list1, list2, fillvalue='N/A'))
print("zip_longest:", with_longest)
# [(1, 'a'), (2, 'b'), (3, 'c'), (4, 'N/A'), (5, 'N/A')]

# ตัวอย่างจริง: transpose matrix
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
transposed = list(map(list, zip(*matrix)))
print("transposed:", transposed)
# [[1, 4, 7], [2, 5, 8], [3, 6, 9]]

# ใช้ zip เพื่อเปรียบเทียบ 2 lists
before = [100, 200, 300]
after  = [120, 180, 350]
changes = [(b, a, a - b) for b, a in zip(before, after)]
print("changes:", changes)  # [(100, 120, 20), (200, 180, -20), (300, 350, 50)]
```

---

## 7. enumerate() - ลูปพร้อม Index

```python
# enumerate(iterable, start=0) - คืน (index, value) pairs

fruits = ['แอปเปิล', 'กล้วย', 'ส้ม', 'มะม่วง']

# วิธีเก่า (ไม่แนะนำ)
for i in range(len(fruits)):
    print(f"{i}: {fruits[i]}")

# วิธี enumerate (แนะนำ)
for i, fruit in enumerate(fruits):
    print(f"{i}: {fruit}")

# เริ่มจาก 1
for num, fruit in enumerate(fruits, start=1):
    print(f"{num}. {fruit}")
# 1. แอปเปิล
# 2. กล้วย
# ...

# ใช้ enumerate เพื่อหา index ของค่าที่ต้องการ
def find_index(lst, value):
    """หา index ของค่าใน list"""
    for i, item in enumerate(lst):
        if item == value:
            return i
    return -1

print(find_index(fruits, 'ส้ม'))  # 2
print(find_index(fruits, 'ทุเรียน'))  # -1

# สร้าง dict จาก enumerate
word = "PYTHON"
char_positions = {char: i for i, char in enumerate(word)}
print(char_positions)  # {'P': 0, 'Y': 1, 'T': 2, 'H': 3, 'O': 4, 'N': 5}

# ตัวอย่างจริง: ทำ numbered list สำหรับ menu
def display_menu(items, title="เมนู"):
    print(f"\n{'='*30}")
    print(f"  {title}")
    print(f"{'='*30}")
    for num, item in enumerate(items, 1):
        print(f"  {num}. {item}")
    print(f"{'='*30}")
    return input("เลือก: ")

menu = ['ข้าวผัด', 'ผัดกะเพรา', 'ต้มยำ', 'ส้มตำ', 'ออก']
# choice = display_menu(menu, "อาหารประจำวัน")

# enumerate กับ list comprehension
indexed_evens = [(i, x) for i, x in enumerate(range(10)) if x % 2 == 0]
print(indexed_evens)  # [(0, 0), (2, 2), (4, 4), (6, 6), (8, 8)]
```

---

## 8. sorted() with key

```python
# sorted(iterable, key=None, reverse=False) - คืน sorted list ใหม่
# list.sort() - sort in-place

numbers = [3, 1, 4, 1, 5, 9, 2, 6, 5]

# sort ธรรมดา
print(sorted(numbers))           # [1, 1, 2, 3, 4, 5, 5, 6, 9]
print(sorted(numbers, reverse=True))  # [9, 6, 5, 5, 4, 3, 2, 1, 1]

# sort by key
words = ['banana', 'apple', 'cherry', 'date', 'elderberry']
print(sorted(words))                          # alphabetical
print(sorted(words, key=len))                 # by length
print(sorted(words, key=lambda w: w[-1]))     # by last character

# sort objects
students = [
    {'name': 'อลิสา', 'age': 25, 'gpa': 3.8},
    {'name': 'บอม', 'age': 22, 'gpa': 3.5},
    {'name': 'ชาลี', 'age': 28, 'gpa': 3.9},
    {'name': 'ดาว', 'age': 22, 'gpa': 3.7},
]

# sort by age
by_age = sorted(students, key=lambda s: s['age'])
print("by age:", [s['name'] for s in by_age])
# ['บอม', 'ดาว', 'อลิสา', 'ชาลี']

# sort by GPA (descending)
by_gpa = sorted(students, key=lambda s: s['gpa'], reverse=True)
print("by gpa:", [s['name'] for s in by_gpa])
# ['ชาลี', 'อลิสา', 'ดาว', 'บอม']

# sort by หลาย criteria
by_age_then_gpa = sorted(
    students,
    key=lambda s: (s['age'], -s['gpa'])  # age ascending, gpa descending
)
for s in by_age_then_gpa:
    print(f"  {s['name']}: อายุ {s['age']}, GPA {s['gpa']}")

# operator.itemgetter - เร็วกว่า lambda
from operator import itemgetter, attrgetter

by_age_fast = sorted(students, key=itemgetter('age'))
print("fast by age:", [s['name'] for s in by_age_fast])

# sort Tuples
data = [(2, 'b'), (1, 'c'), (3, 'a')]
print(sorted(data))                    # [(1, 'c'), (2, 'b'), (3, 'a')]
print(sorted(data, key=lambda x: x[1]))  # [(3, 'a'), (2, 'b'), (1, 'c')]
```

---

## 9. all() และ any()

```python
# all(iterable) - True ถ้าทุก element เป็น True
# any(iterable) - True ถ้ามีอย่างน้อย 1 element เป็น True

numbers = [2, 4, 6, 8, 10]
print(all(x % 2 == 0 for x in numbers))  # True - ทุกตัวเป็นคู่
print(any(x > 9 for x in numbers))        # True - มีตัว > 9

mixed = [2, 3, 4, 6]
print(all(x % 2 == 0 for x in mixed))    # False - 3 ไม่ใช่เลขคู่
print(any(x % 2 != 0 for x in mixed))   # True - 3 เป็นเลขคี่

# all() กับ empty iterable คืน True (vacuous truth)
print(all([]))  # True
print(any([]))  # False

# ตัวอย่างจริง: Validation
def validate_user(user):
    """ตรวจสอบข้อมูล user"""
    validations = [
        (bool(user.get('name')), "ต้องมีชื่อ"),
        (len(user.get('name', '')) >= 2, "ชื่อต้องมีอย่างน้อย 2 ตัวอักษร"),
        (bool(user.get('email')), "ต้องมี email"),
        ('@' in user.get('email', ''), "email ไม่ถูกต้อง"),
        (user.get('age', 0) >= 18, "ต้องอายุ 18 ปีขึ้นไป"),
    ]
    
    errors = [msg for is_valid, msg in validations if not is_valid]
    return len(errors) == 0, errors

user1 = {'name': 'สมชาย', 'email': 'somchai@email.com', 'age': 25}
user2 = {'name': 'A', 'email': 'invalid', 'age': 16}

valid1, errors1 = validate_user(user1)
valid2, errors2 = validate_user(user2)

print(f"User1 valid: {valid1}")      # True
print(f"User2 valid: {valid2}")      # False
print(f"Errors: {errors2}")

# any() เพื่อตรวจสอบ permissions
user_perms = {'read', 'write'}
required_any = {'admin', 'write', 'superuser'}
required_all = {'read', 'write', 'delete'}

has_any = any(p in user_perms for p in required_any)
has_all = all(p in user_perms for p in required_all)

print(f"Has any permission: {has_any}")   # True (has 'write')
print(f"Has all permissions: {has_all}")  # False (no 'delete')
```

---

## 10. functools Module

```python
from functools import partial, reduce, lru_cache, wraps, total_ordering
import time

# ============================================================
# 10.1 partial() - สร้างฟังก์ชันที่ fix บาง arguments ล่วงหน้า
# ============================================================
def power(base, exponent):
    return base ** exponent

# สร้างฟังก์ชันใหม่ที่ fix exponent
square = partial(power, exponent=2)
cube = partial(power, exponent=3)
double = partial(power, exponent=1)

print(square(5))  # 25
print(cube(3))    # 27

# partial กับ print()
print_error = partial(print, "[ERROR]", end="\n", flush=True)
print_info = partial(print, "[INFO]", end="\n")

print_error("การเชื่อมต่อล้มเหลว")  # [ERROR] การเชื่อมต่อล้มเหลว
print_info("เริ่มระบบแล้ว")         # [INFO] เริ่มระบบแล้ว

# partial สำหรับ database operations
def db_query(connection, table, condition=None):
    query = f"SELECT * FROM {table}"
    if condition:
        query += f" WHERE {condition}"
    return query

# สร้าง query functions สำหรับแต่ละ table
# query_users = partial(db_query, connection=db_conn, table='users')
# query_orders = partial(db_query, connection=db_conn, table='orders')

# ============================================================
# 10.2 lru_cache() - Cache ผลลัพธ์
# ============================================================
@lru_cache(maxsize=128)
def fibonacci(n):
    """Fibonacci ที่มี cache"""
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

# ครั้งแรก: คำนวณจริง
start = time.time()
result = fibonacci(50)
t1 = time.time() - start

# ครั้งที่ 2: จาก cache
start = time.time()
result2 = fibonacci(50)
t2 = time.time() - start

print(f"fibonacci(50) = {result}")
print(f"ครั้งแรก: {t1*1000:.3f}ms")
print(f"จาก cache: {t2*1000:.3f}ms")

# ดู cache info
print(fibonacci.cache_info())
# CacheInfo(hits=49, misses=51, maxsize=128, currsize=51)

# ============================================================
# 10.3 total_ordering - เติม comparison methods อัตโนมัติ
# ============================================================
@total_ordering
class Student:
    def __init__(self, name, gpa):
        self.name = name
        self.gpa = gpa
    
    def __eq__(self, other):
        return self.gpa == other.gpa
    
    def __lt__(self, other):
        return self.gpa < other.gpa
    # @total_ordering จะสร้าง __gt__, __le__, __ge__ ให้อัตโนมัติ

s1 = Student("อลิสา", 3.8)
s2 = Student("บอม", 3.5)
s3 = Student("ชาลี", 3.8)

print(s1 > s2)    # True
print(s1 >= s3)   # True
print(s2 <= s3)   # True
print(sorted([s1, s2, s3], key=lambda s: s.gpa))
```

---

## 11. Closures และ Higher-Order Functions

```python
# Closure = function ที่จำ variables จาก scope ภายนอก

def make_counter(start=0, step=1):
    """สร้าง counter function"""
    count = [start]  # ใช้ list เพื่อ mutability
    
    def counter():
        current = count[0]
        count[0] += step
        return current
    
    return counter

counter = make_counter(0, 2)
print(counter())  # 0
print(counter())  # 2
print(counter())  # 4

# แต่ละ counter เป็น instance แยกกัน
counter_a = make_counter(1, 1)
counter_b = make_counter(100, 10)

print(counter_a())  # 1
print(counter_b())  # 100
print(counter_a())  # 2
print(counter_b())  # 110

# Higher-Order Functions
def memoize(func):
    """Cache ผลลัพธ์ของฟังก์ชัน"""
    cache = {}
    
    @wraps(func)
    def wrapper(*args):
        if args not in cache:
            cache[args] = func(*args)
        return cache[args]
    
    wrapper.cache = cache  # expose cache
    return wrapper

@memoize
def expensive_calculation(n):
    """คำนวณที่ใช้เวลานาน"""
    time.sleep(0.01)  # simulate delay
    return n * n

import time
start = time.time()
for _ in range(5):
    expensive_calculation(10)  # คำนวณครั้งแรก จาก cache ที่เหลือ
print(f"เวลา: {(time.time()-start)*1000:.1f}ms")  # ~10ms ไม่ใช่ 50ms

print("Cache:", expensive_calculation.cache)

# Function composition
def compose(*funcs):
    """สร้าง composition ของ functions"""
    def composed(x):
        result = x
        for func in reversed(funcs):
            result = func(result)
        return result
    return composed

def add_tax(price):
    return price * 1.07

def apply_discount(price):
    return price * 0.9

def round_price(price):
    return round(price, 2)

final_price = compose(round_price, add_tax, apply_discount)
print(f"ราคาสุดท้าย: {final_price(1000)}")  # round_price(add_tax(apply_discount(1000)))
```

---

## 12. ตัวอย่างโปรแกรมจริง: Data Pipeline

```python
from functools import reduce
import json

# ข้อมูลยอดขาย
sales_data = [
    {'product': 'Laptop', 'price': 25000, 'qty': 3, 'region': 'BKK', 'month': 1},
    {'product': 'Mouse', 'price': 500, 'qty': 15, 'region': 'CNX', 'month': 1},
    {'product': 'Keyboard', 'price': 1200, 'qty': 8, 'region': 'BKK', 'month': 2},
    {'product': 'Monitor', 'price': 8000, 'qty': 2, 'region': 'CNX', 'month': 2},
    {'product': 'Laptop', 'price': 26000, 'qty': 5, 'region': 'BKK', 'month': 3},
    {'product': 'Headset', 'price': 2500, 'qty': 6, 'region': 'CNX', 'month': 3},
    {'product': 'Webcam', 'price': 1800, 'qty': 4, 'region': 'BKK', 'month': 1},
]

# === Functional Pipeline ===
# Step 1: เพิ่ม total
add_total = lambda s: {**s, 'total': s['price'] * s['qty']}

# Step 2: กรองเฉพาะ BKK
is_bkk = lambda s: s['region'] == 'BKK'

# Step 3: เรียงตาม total
by_total = lambda s: s['total']

# Step 4: สร้าง summary
def summarize(sales):
    items = sorted(
        filter(is_bkk, map(add_total, sales)),
        key=by_total,
        reverse=True
    )
    return items

bkk_sales = summarize(sales_data)
print("ยอดขาย BKK (เรียงจากมากไปน้อย):")
for sale in bkk_sales:
    print(f"  {sale['product']:<12} {sale['total']:>10,} บาท")

# คำนวณสถิติ
totals = list(map(lambda s: s['total'], map(add_total, sales_data)))
print(f"\nสถิติรวม:")
print(f"  รวมทั้งหมด: {sum(totals):,} บาท")
print(f"  สูงสุด:     {max(totals):,} บาท")
print(f"  ต่ำสุด:     {min(totals):,} บาท")
print(f"  เฉลี่ย:     {sum(totals)/len(totals):,.0f} บาท")

# Group by region
from itertools import groupby
sales_with_total = sorted(
    map(add_total, sales_data),
    key=lambda s: s['region']
)

print("\nสรุปตาม region:")
for region, group in groupby(sales_with_total, key=lambda s: s['region']):
    group_list = list(group)
    region_total = sum(s['total'] for s in group_list)
    print(f"  {region}: {len(group_list)} รายการ, รวม {region_total:,} บาท")
```

---

## สรุป Part 020

✅ **Functional Programming** เน้น pure functions, immutable data, higher-order functions  
✅ **lambda** ใช้สำหรับ expressions สั้นๆ ที่ใช้ครั้งเดียว  
✅ **map()** แปลง iterable ทีละ element  
✅ **filter()** กรอง elements ที่ผ่านเงื่อนไข  
✅ **reduce()** รวม iterable เป็นค่าเดียว  
✅ **zip()** จับคู่ elements จากหลาย iterables  
✅ **enumerate()** ได้ทั้ง index และ value ใน loop  
✅ **sorted()** กับ key function สำหรับ custom sorting  
✅ **all()/any()** ตรวจสอบ conditions บน iterable  
✅ **functools.partial** สร้างฟังก์ชันใหม่โดย fix บาง arguments  
✅ **functools.lru_cache** cache ผลลัพธ์อัตโนมัติ  

---

## ➡️ ถัดไป: Part 021 - Regular Expressions

*Part 020/100+ | Python Course - Beginner to World-Class*
