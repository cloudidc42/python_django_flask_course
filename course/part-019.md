# Part 019: Generators (เจนเนอเรเตอร์)
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้
- เข้าใจความแตกต่างระหว่าง Generator กับ List ธรรมดา
- ใช้ `yield` เพื่อสร้าง Generator Function
- สร้าง Generator Expression
- ใช้ `next()` และ `iter()`
- ใช้ `send()` เพื่อส่งค่าเข้า Generator
- สร้าง Infinite Generators
- ใช้ `itertools` module อย่างมีประสิทธิภาพ

---

## 1. Generator คืออะไร?

Generator คือฟังก์ชันพิเศษที่สามารถ "หยุดชั่วคราว" และ "ดำเนินต่อ" ได้ โดยใช้ `yield`

### ทำไมต้องใช้ Generator?

```python
# ปัญหา: สร้าง list ขนาดใหญ่ใช้ RAM มาก
import sys

# วิธีปกติ - สร้าง list ทั้งหมดในหน่วยความจำ
def get_numbers_list(n):
    """คืน list ของตัวเลข 0 ถึง n-1"""
    return [i * i for i in range(n)]

# วิธี Generator - สร้างทีละตัว ประหยัด RAM
def get_numbers_gen(n):
    """Generator ที่สร้างตัวเลขทีละตัว"""
    for i in range(n):
        yield i * i

# เปรียบเทียบ memory usage
numbers_list = get_numbers_list(1_000_000)
numbers_gen = get_numbers_gen(1_000_000)

print(f"List size: {sys.getsizeof(numbers_list):,} bytes")      # ~8 MB
print(f"Generator size: {sys.getsizeof(numbers_gen):,} bytes")  # ~112 bytes

# Generator คือ object ที่ยัง "ไม่ได้คำนวณ"
print(type(numbers_gen))  # <class 'generator'>
```

---

## 2. yield - หัวใจของ Generator

```python
# yield ทำให้ฟังก์ชันกลายเป็น generator
def simple_generator():
    """Generator ง่ายๆ ที่ yield ค่าทีละตัว"""
    print("เริ่มต้น...")
    yield 1          # หยุดที่นี่ คืนค่า 1
    
    print("กลับมาต่อ...")
    yield 2          # หยุดที่นี่ คืนค่า 2
    
    print("กลับมาอีกครั้ง...")
    yield 3          # หยุดที่นี่ คืนค่า 3
    
    print("จบแล้ว!")

# สร้าง generator object
gen = simple_generator()
print(type(gen))     # <class 'generator'>

# เรียก next() เพื่อดึงค่าถัดไป
print(next(gen))  # เริ่มต้น... -> 1
print(next(gen))  # กลับมาต่อ... -> 2
print(next(gen))  # กลับมาอีกครั้ง... -> 3

# เรียก next() อีกครั้งจะเกิด StopIteration
try:
    print(next(gen))  # จบแล้ว! -> StopIteration
except StopIteration:
    print("Generator หมดแล้ว!")
```

```python
# ใช้ for loop กับ generator (แนะนำ)
def countdown(n):
    """นับถอยหลัง"""
    print(f"เริ่มนับถอยหลังจาก {n}")
    while n > 0:
        yield n
        n -= 1
    print("ปล่อย!")

# for loop จัดการ StopIteration ให้อัตโนมัติ
for num in countdown(5):
    print(f"  {num}...")

# Output:
# เริ่มนับถอยหลังจาก 5
#   5...
#   4...
#   3...
#   2...
#   1...
# ปล่อย!
```

---

## 3. Generator Functions ในชีวิตจริง

```python
# ตัวอย่าง: อ่านไฟล์ใหญ่ทีละบรรทัด
def read_large_file(filepath):
    """
    อ่านไฟล์ขนาดใหญ่ทีละบรรทัดโดยไม่โหลดทั้งไฟล์ขึ้น RAM
    """
    with open(filepath, 'r', encoding='utf-8') as f:
        for line in f:
            yield line.strip()

# ใช้งาน
# for line in read_large_file('huge_log.txt'):
#     process(line)  # ประมวลผลทีละบรรทัด

# ตัวอย่าง: Fibonacci Generator
def fibonacci():
    """สร้างตัวเลข Fibonacci ไม่สิ้นสุด"""
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

# ดึง 10 ตัวแรก
fib = fibonacci()
first_10 = [next(fib) for _ in range(10)]
print(first_10)  # [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

# ตัวอย่าง: สร้างเลขเฉพาะ
def primes():
    """Generator สำหรับเลขเฉพาะ"""
    def is_prime(n):
        if n < 2:
            return False
        for i in range(2, int(n**0.5) + 1):
            if n % i == 0:
                return False
        return True
    
    n = 2
    while True:
        if is_prime(n):
            yield n
        n += 1

# ดึง 10 เลขเฉพาะแรก
prime_gen = primes()
print([next(prime_gen) for _ in range(10)])
# [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]
```

```python
# ตัวอย่าง: Pagination Generator
def paginate(data, page_size=10):
    """แบ่ง data เป็นหน้าๆ"""
    for i in range(0, len(data), page_size):
        yield data[i:i + page_size]

# ใช้งาน
users = list(range(1, 53))  # 52 users
for page_num, page in enumerate(paginate(users, 10), 1):
    print(f"หน้า {page_num}: {page}")

# Output:
# หน้า 1: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
# หน้า 2: [11, 12, 13, 14, 15, 16, 17, 18, 19, 20]
# ...

# ตัวอย่าง: Walking Directory Tree
import os

def walk_files(directory, extension=None):
    """เดิน directory tree และ yield ไฟล์"""
    for root, dirs, files in os.walk(directory):
        for filename in files:
            if extension is None or filename.endswith(extension):
                yield os.path.join(root, filename)

# ใช้งาน
# for py_file in walk_files('.', '.py'):
#     print(py_file)
```

---

## 4. Generator Expressions

```python
# Generator Expression - เหมือน List Comprehension แต่ใช้ ()
# ประหยัดหน่วยความจำ เพราะไม่สร้างทั้งหมดในคราวเดียว

# List Comprehension - สร้างทั้งหมดทันที
squares_list = [x**2 for x in range(1000000)]  # ใช้ ~8MB RAM

# Generator Expression - สร้างทีละตัวเมื่อถูกขอ
squares_gen = (x**2 for x in range(1000000))   # ใช้ ~120 bytes

print(type(squares_list))  # <class 'list'>
print(type(squares_gen))   # <class 'generator'>

# ใช้ใน for loop
total = sum(x**2 for x in range(100))  # ส่ง generator expression ให้ sum()
print(total)  # 328350

# กรองข้อมูล
even_squares = list(x**2 for x in range(20) if x % 2 == 0)
print(even_squares)  # [0, 4, 16, 36, 64, 100, 144, 196, 256, 324]

# ซ้อน generator expressions
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
flat = list(num for row in matrix for num in row)
print(flat)  # [1, 2, 3, 4, 5, 6, 7, 8, 9]

# ใช้กับ any() และ all()
numbers = [2, 4, 6, 7, 10, 12]
all_even = all(n % 2 == 0 for n in numbers)
has_odd = any(n % 2 != 0 for n in numbers)
print(f"ทั้งหมดเป็นคู่: {all_even}")  # False
print(f"มีเลขคี่: {has_odd}")          # True
```

---

## 5. next() และ iter()

```python
# next() - ดึงค่าถัดไปจาก iterator
gen = (x**2 for x in range(5))

print(next(gen))  # 0
print(next(gen))  # 1
print(next(gen))  # 4

# next() พร้อม default value
print(next(gen, "หมดแล้ว"))  # 9
print(next(gen, "หมดแล้ว"))  # 16
print(next(gen, "หมดแล้ว"))  # หมดแล้ว (ไม่เกิด StopIteration)
print(next(gen, "หมดแล้ว"))  # หมดแล้ว

# iter() - แปลง iterable ให้เป็น iterator
my_list = [10, 20, 30, 40]
it = iter(my_list)  # สร้าง iterator จาก list

print(next(it))  # 10
print(next(it))  # 20

# iter() กับ sentinel value
# iter(callable, sentinel) - เรียก callable จนกว่าจะได้ sentinel
import random

def roll_dice():
    return random.randint(1, 6)

# หยุดเมื่อได้ 6
rolls = list(iter(roll_dice, 6))
print(f"ลูกเต๋าก่อนได้ 6: {rolls}")

# ตัวอย่างเชิงปฏิบัติ: อ่าน input จนกว่าจะว่าง
def get_user_input():
    return input("ใส่ข้อความ (กด Enter เพื่อหยุด): ")

# lines = list(iter(get_user_input, ''))  # หยุดเมื่อ input ว่าง

# Manual iteration
class NumberRange:
    """Class ที่สนับสนุน iteration"""
    def __init__(self, start, end):
        self.start = start
        self.end = end
    
    def __iter__(self):
        """คืน iterator"""
        return iter(range(self.start, self.end + 1))

numbers = NumberRange(1, 5)
for n in numbers:
    print(n, end=" ")  # 1 2 3 4 5
print()

# สามารถ iterate หลายครั้ง
for n in numbers:
    print(n, end=" ")  # 1 2 3 4 5
```

---

## 6. send() - ส่งค่าเข้า Generator

```python
# send() ให้เราส่งค่ากลับเข้าไปใน generator
def accumulator():
    """Generator ที่รับค่าและสะสม"""
    total = 0
    while True:
        value = yield total  # yield ส่งออก และรับค่าจาก send()
        if value is None:
            break
        total += value
        print(f"  รับ {value}, รวม = {total}")

# ใช้งาน
acc = accumulator()
next(acc)  # เริ่ม generator (ต้อง next() ก่อนใช้ send())

print(acc.send(10))   # รับ 10, รวม = 10  -> 10
print(acc.send(20))   # รับ 20, รวม = 30  -> 30
print(acc.send(5))    # รับ 5, รวม = 35   -> 35

# ตัวอย่างเพิ่มเติม: Calculator Generator
def calculator():
    """Generator ที่ทำหน้าที่เครื่องคิดเลข"""
    result = 0
    print(f"เริ่มต้นด้วย {result}")
    
    while True:
        expression = yield result
        if expression is None:
            return
        
        try:
            # รับ tuple (operator, number)
            op, num = expression
            if op == '+':
                result += num
            elif op == '-':
                result -= num
            elif op == '*':
                result *= num
            elif op == '/':
                result /= num
            print(f"  {op} {num} = {result}")
        except (TypeError, ValueError) as e:
            print(f"  ข้อผิดพลาด: {e}")

calc = calculator()
next(calc)  # เริ่ม

calc.send(('+', 100))   # + 100 = 100
calc.send(('*', 2))     # * 2 = 200
calc.send(('-', 50))    # - 50 = 150
print(f"ผลลัพธ์สุดท้าย: {calc.send(('/', 3))}")  # ÷ 3 = 50.0

# close() และ throw() 
def my_gen():
    try:
        yield 1
        yield 2
        yield 3
    except GeneratorExit:
        print("Generator ถูกปิด!")
    except ValueError as e:
        print(f"ได้รับ ValueError: {e}")
        yield "recovered"

g = my_gen()
print(next(g))  # 1
g.throw(ValueError, "test error")  # ได้รับ ValueError: test error
g.close()  # Generator ถูกปิด!
```

---

## 7. Infinite Generators

```python
# Generator ที่ไม่มีวันสิ้นสุด
def natural_numbers():
    """จำนวนธรรมชาติ 1, 2, 3, ..."""
    n = 1
    while True:
        yield n
        n += 1

def cycle_colors():
    """วนเวียนสี"""
    colors = ['แดง', 'เขียว', 'น้ำเงิน', 'เหลือง']
    idx = 0
    while True:
        yield colors[idx % len(colors)]
        idx += 1

# ใช้งาน - ต้องมีเงื่อนไขหยุด!
nat = natural_numbers()
first_5 = [next(nat) for _ in range(5)]
print(first_5)  # [1, 2, 3, 4, 5]

colors = cycle_colors()
palette = [next(colors) for _ in range(7)]
print(palette)  # ['แดง', 'เขียว', 'น้ำเงิน', 'เหลือง', 'แดง', 'เขียว', 'น้ำเงิน']

# ดึงค่าจนถึงเงื่อนไข
def take_while(gen, condition):
    """ดึงค่าจาก generator จนกว่า condition จะเป็น False"""
    for item in gen:
        if not condition(item):
            break
        yield item

# ดึงเลขธรรมชาติที่ < 10
small = list(take_while(natural_numbers(), lambda x: x < 10))
print(small)  # [1, 2, 3, 4, 5, 6, 7, 8, 9]

# ตัวอย่างเชิงปฏิบัติ: ID Generator
def id_generator(prefix='USR'):
    """สร้าง unique IDs"""
    n = 1
    while True:
        yield f"{prefix}-{n:05d}"
        n += 1

user_ids = id_generator('USR')
order_ids = id_generator('ORD')

print(next(user_ids))   # USR-00001
print(next(user_ids))   # USR-00002
print(next(order_ids))  # ORD-00001
print(next(user_ids))   # USR-00003 (สืบเนื่องจากเดิม)
```

---

## 8. itertools Module

`itertools` เป็น module มาตรฐานที่มี generator utilities ที่มีประสิทธิภาพสูง

```python
import itertools

# ============================================================
# 8.1 itertools.count() - นับแบบ infinite
# ============================================================
counter = itertools.count(start=10, step=5)
first_5 = list(itertools.islice(counter, 5))
print("count:", first_5)  # [10, 15, 20, 25, 30]

# ใช้กับ zip() เพื่อ enumerate แบบกำหนด start
items = ['a', 'b', 'c', 'd']
numbered = list(zip(itertools.count(1), items))
print("numbered:", numbered)  # [(1, 'a'), (2, 'b'), (3, 'c'), (4, 'd')]

# ============================================================
# 8.2 itertools.cycle() - วนซ้ำ
# ============================================================
seasons = itertools.cycle(['ฤดูร้อน', 'ฤดูฝน', 'ฤดูหนาว'])
year_seasons = list(itertools.islice(seasons, 9))
print("seasons:", year_seasons)
# ['ฤดูร้อน', 'ฤดูฝน', 'ฤดูหนาว', 'ฤดูร้อน', 'ฤดูฝน', 'ฤดูหนาว', 'ฤดูร้อน', 'ฤดูฝน', 'ฤดูหนาว']

# ใช้ cycle() สำหรับ round-robin
def round_robin(iterables):
    """สลับวนไปมาระหว่าง iterables"""
    nexts = itertools.cycle(iter(it) for it in iterables)
    pending = len(iterables)
    while pending:
        try:
            for next_fn in nexts:
                yield next(next_fn)
        except StopIteration:
            pending -= 1
            nexts = itertools.cycle(
                itertools.islice(nexts, pending)
            )

result = list(round_robin(['ABC', 'D', 'EF']))
print("round_robin:", result)  # ['A', 'D', 'E', 'B', 'F', 'C']
```

```python
import itertools

# ============================================================
# 8.3 itertools.chain() - ต่อ iterables
# ============================================================
list1 = [1, 2, 3]
list2 = [4, 5, 6]
list3 = [7, 8, 9]

combined = list(itertools.chain(list1, list2, list3))
print("chain:", combined)  # [1, 2, 3, 4, 5, 6, 7, 8, 9]

# chain.from_iterable() - สำหรับ nested iterables
nested = [[1, 2], [3, 4], [5, 6]]
flat = list(itertools.chain.from_iterable(nested))
print("chain.from_iterable:", flat)  # [1, 2, 3, 4, 5, 6]

# ใช้ chain เพื่อ flatten dictionary values
data = {'a': [1, 2], 'b': [3, 4], 'c': [5, 6]}
all_values = list(itertools.chain.from_iterable(data.values()))
print("dict values flattened:", all_values)  # [1, 2, 3, 4, 5, 6]

# ============================================================
# 8.4 itertools.islice() - ตัด slice จาก iterator
# ============================================================
gen = (x**2 for x in itertools.count(1))  # infinite generator

# เหมือน slice แต่ใช้กับ iterator ได้
first_10 = list(itertools.islice(gen, 10))
print("islice first 10:", first_10)

# islice(iterable, start, stop, step)
gen2 = itertools.count(0)
every_other = list(itertools.islice(gen2, 2, 20, 3))
print("islice with step:", every_other)  # [2, 5, 8, 11, 14, 17]
```

```python
import itertools

# ============================================================
# 8.5 itertools.product() - Cartesian Product
# ============================================================
# เทียบเท่ากับ nested for loops

colors = ['แดง', 'เขียว']
sizes = ['S', 'M', 'L']

# แบบปกติ (nested loops)
products_normal = []
for c in colors:
    for s in sizes:
        products_normal.append((c, s))

# แบบ itertools (เร็วกว่า)
products_iter = list(itertools.product(colors, sizes))
print("product:", products_iter)
# [('แดง', 'S'), ('แดง', 'M'), ('แดง', 'L'), ('เขียว', 'S'), ('เขียว', 'M'), ('เขียว', 'L')]

# repeat parameter
dice_rolls = list(itertools.product(range(1, 7), repeat=2))
print(f"dice combinations: {len(dice_rolls)}")  # 36

# ============================================================
# 8.6 itertools.combinations() - การจัดหมู่
# ============================================================
team = ['อลิสา', 'บอม', 'ชาลี', 'ดาว']

# เลือก 2 คน (ไม่สนลำดับ, ไม่ซ้ำ)
pairs = list(itertools.combinations(team, 2))
print(f"\nจับคู่ได้ {len(pairs)} คู่:")
for pair in pairs:
    print(f"  {pair[0]} + {pair[1]}")

# combinations_with_replacement - ซ้ำได้
coins = list(itertools.combinations_with_replacement([1, 5, 10], 2))
print("\nคู่เหรียญ (ซ้ำได้):", coins)

# ============================================================
# 8.7 itertools.permutations() - การเรียงสับเปลี่ยน
# ============================================================
letters = ['A', 'B', 'C']
perms = list(itertools.permutations(letters))
print(f"\nเรียงลำดับ {len(perms)} แบบ:")
for p in perms:
    print("  " + "".join(p))
# ABC, ACB, BAC, BCA, CAB, CBA
```

```python
import itertools

# ============================================================
# 8.8 Filtering itertools
# ============================================================

# itertools.filterfalse() - กรองค่าที่เป็น False
numbers = range(1, 11)
odds = list(itertools.filterfalse(lambda x: x % 2 == 0, numbers))
print("odd numbers:", odds)  # [1, 3, 5, 7, 9]

# itertools.compress() - เลือกตาม mask
data = ['a', 'b', 'c', 'd', 'e']
mask = [True, False, True, False, True]
selected = list(itertools.compress(data, mask))
print("compressed:", selected)  # ['a', 'c', 'e']

# itertools.takewhile() - รับจนเงื่อนไขเป็น False
numbers = [1, 3, 5, 6, 7, 9]
# รับเลขคี่ต่อเนื่องจากต้น
odds_from_start = list(itertools.takewhile(lambda x: x % 2 != 0, numbers))
print("takewhile odd:", odds_from_start)  # [1, 3, 5]

# itertools.dropwhile() - ข้ามจนเงื่อนไขเป็น False
remaining = list(itertools.dropwhile(lambda x: x % 2 != 0, numbers))
print("dropwhile odd:", remaining)  # [6, 7, 9]

# ============================================================
# 8.9 itertools.groupby() - จัดกลุ่ม
# ============================================================
from itertools import groupby
from operator import itemgetter

students = [
    {'name': 'อลิสา', 'grade': 'A'},
    {'name': 'บอม', 'grade': 'B'},
    {'name': 'ชาลี', 'grade': 'A'},
    {'name': 'ดาว', 'grade': 'C'},
    {'name': 'เอก', 'grade': 'B'},
    {'name': 'ฟ้า', 'grade': 'A'},
]

# ต้อง sort ก่อน groupby
students.sort(key=itemgetter('grade'))

print("จัดกลุ่มนักเรียนตามเกรด:")
for grade, group in groupby(students, key=itemgetter('grade')):
    names = [s['name'] for s in group]
    print(f"  เกรด {grade}: {', '.join(names)}")

# Output:
# จัดกลุ่มนักเรียนตามเกรด:
#   เกรด A: อลิสา, ชาลี, ฟ้า
#   เกรด B: บอม, เอก
#   เกรด C: ดาว
```

---

## 9. Generator Pipelines

```python
# สร้าง pipeline ของ generators เพื่อประมวลผลข้อมูล
import itertools

def read_data(source):
    """Stage 1: อ่านข้อมูล"""
    for item in source:
        yield item

def filter_valid(data, min_val=0):
    """Stage 2: กรองข้อมูลที่ valid"""
    for item in data:
        if item >= min_val:
            yield item

def transform(data, multiplier=2):
    """Stage 3: แปลงข้อมูล"""
    for item in data:
        yield item * multiplier

def format_output(data, prefix="Value"):
    """Stage 4: จัดรูปแบบ output"""
    for item in data:
        yield f"{prefix}: {item}"

# สร้าง pipeline
raw_data = [5, -2, 3, -1, 8, 0, 4, -3, 7]

pipeline = format_output(
    transform(
        filter_valid(
            read_data(raw_data),
            min_val=1
        ),
        multiplier=3
    ),
    prefix="ผลลัพธ์"
)

for result in pipeline:
    print(result)

# Output:
# ผลลัพธ์: 15
# ผลลัพธ์: 9
# ผลลัพธ์: 24
# ผลลัพธ์: 12
# ผลลัพธ์: 21
```

```python
# ตัวอย่างจริง: ETL Pipeline
import csv
import json
from io import StringIO

def read_csv_data(csv_text):
    """อ่านข้อมูล CSV"""
    reader = csv.DictReader(StringIO(csv_text))
    for row in reader:
        yield row

def clean_data(rows):
    """ทำความสะอาดข้อมูล"""
    for row in rows:
        # แปลง type และทำความสะอาด
        yield {
            'name': row['name'].strip().title(),
            'age': int(row['age']),
            'salary': float(row['salary'].replace(',', '')),
            'department': row['department'].strip()
        }

def filter_employees(rows, min_salary=30000):
    """กรองพนักงานที่เงินเดือนเกินกำหนด"""
    for row in rows:
        if row['salary'] >= min_salary:
            yield row

def add_tax(rows, tax_rate=0.07):
    """คำนวณภาษี"""
    for row in rows:
        row['tax'] = round(row['salary'] * tax_rate, 2)
        row['net_salary'] = round(row['salary'] * (1 - tax_rate), 2)
        yield row

# ข้อมูล CSV ตัวอย่าง
csv_data = """name,age,salary,department
  สมชาย  ,25,25000,IT
อรอุมา,30,45000.0,HR
 พีรพล ,28, 55,000,Engineering
กนกวรรณ,35,38000,Marketing
ธีรยุทธ,22,18000,IT"""

# รัน pipeline
pipeline = add_tax(
    filter_employees(
        clean_data(
            read_csv_data(csv_data)
        ),
        min_salary=30000
    )
)

print("พนักงานที่เงินเดือน >= 30,000:")
print("-" * 60)
for emp in pipeline:
    print(f"{emp['name']:<15} เงินเดือน: {emp['salary']:>10,.0f}"
          f"  ภาษี: {emp['tax']:>8,.0f}"
          f"  รับจริง: {emp['net_salary']:>10,.0f}")
```

---

## 10. yield from - Delegating to Sub-generators

```python
# yield from ส่งต่อ iteration ให้ sub-generator
def inner_gen():
    yield 1
    yield 2
    yield 3

def outer_gen_without():
    """แบบไม่ใช้ yield from"""
    for item in inner_gen():
        yield item

def outer_gen_with():
    """แบบใช้ yield from (แนะนำ)"""
    yield from inner_gen()  # สั้นกว่า

# ผลเหมือนกัน
print(list(outer_gen_without()))  # [1, 2, 3]
print(list(outer_gen_with()))     # [1, 2, 3]

# yield from กับหลาย iterables
def combined():
    yield from range(1, 4)        # 1, 2, 3
    yield from ['a', 'b', 'c']    # a, b, c
    yield from (x**2 for x in range(1, 4))  # 1, 4, 9

print(list(combined()))  # [1, 2, 3, 'a', 'b', 'c', 1, 4, 9]

# yield from กับ recursive generator (flatten nested)
def flatten(nested):
    """Flatten nested iterable ลึกแค่ไหนก็ได้"""
    for item in nested:
        if isinstance(item, (list, tuple)):
            yield from flatten(item)
        else:
            yield item

data = [1, [2, 3, [4, 5]], 6, [7, [8, [9]]]]
print(list(flatten(data)))  # [1, 2, 3, 4, 5, 6, 7, 8, 9]

# ตัวอย่างเชิงปฏิบัติ: recursive directory walking
def list_all_files(path):
    """List ไฟล์ทั้งหมดแบบ recursive"""
    import os
    for entry in os.scandir(path):
        if entry.is_file():
            yield entry.path
        elif entry.is_dir():
            yield from list_all_files(entry.path)  # recursive

# for f in list_all_files('/some/directory'):
#     print(f)
```

---

## 11. ตัวอย่างโปรแกรมจริง: Log Analyzer

```python
import itertools
import re
from datetime import datetime

# Simulated log data
LOG_DATA = """
2024-01-15 10:30:45 ERROR Database connection failed
2024-01-15 10:30:46 INFO Retry attempt 1
2024-01-15 10:30:47 INFO Retry attempt 2
2024-01-15 10:30:48 ERROR Still failing
2024-01-15 10:31:00 INFO Connection restored
2024-01-15 10:31:05 INFO Processing 100 records
2024-01-15 10:31:10 WARNING Memory usage 85%
2024-01-15 10:31:15 INFO Processing complete
2024-01-15 10:31:20 ERROR File not found: data.csv
2024-01-15 10:31:25 CRITICAL System overload detected
""".strip().split('\n')

def parse_log_line(line):
    """แปลง log line เป็น dict"""
    pattern = r'(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}) (\w+) (.+)'
    match = re.match(pattern, line)
    if match:
        return {
            'timestamp': datetime.strptime(match.group(1), '%Y-%m-%d %H:%M:%S'),
            'level': match.group(2),
            'message': match.group(3)
        }
    return None

def parse_logs(lines):
    """Generator: แปลง log lines"""
    for line in lines:
        parsed = parse_log_line(line.strip())
        if parsed:
            yield parsed

def filter_by_level(logs, levels):
    """Generator: กรองตาม log level"""
    for log in logs:
        if log['level'] in levels:
            yield log

def add_severity_score(logs):
    """Generator: เพิ่ม severity score"""
    scores = {'INFO': 1, 'WARNING': 2, 'ERROR': 3, 'CRITICAL': 4}
    for log in logs:
        log['score'] = scores.get(log['level'], 0)
        yield log

def format_alert(logs, min_score=3):
    """Generator: สร้าง alert messages สำหรับ logs ที่ critical"""
    for log in logs:
        if log['score'] >= min_score:
            yield (f"🚨 [{log['level']}] {log['timestamp'].strftime('%H:%M:%S')}"
                   f" - {log['message']}")

# สร้าง analysis pipeline
pipeline = format_alert(
    add_severity_score(
        parse_logs(LOG_DATA)
    ),
    min_score=3
)

print("=== Log Alerts ===")
for alert in pipeline:
    print(alert)

# สรุปสถิติ
all_logs = list(add_severity_score(parse_logs(LOG_DATA)))

print("\n=== สรุปสถิติ ===")
for level, group in itertools.groupby(
    sorted(all_logs, key=lambda x: x['level']),
    key=lambda x: x['level']
):
    count = sum(1 for _ in group)
    print(f"  {level}: {count} รายการ")
```

---

## 12. เปรียบเทียบ Generator vs List

```python
import sys
import time

# เปรียบเทียบ performance
def measure_time(func, *args):
    start = time.perf_counter()
    result = func(*args)
    end = time.perf_counter()
    return result, (end - start) * 1000

# === Memory Comparison ===
def list_approach(n):
    return [x**2 for x in range(n)]

def gen_approach(n):
    return (x**2 for x in range(n))

n = 1_000_000
lst = list_approach(n)
gen = gen_approach(n)

print("=== Memory Usage ===")
print(f"List:      {sys.getsizeof(lst):>12,} bytes")
print(f"Generator: {sys.getsizeof(gen):>12,} bytes")
print(f"ต่างกัน:    {sys.getsizeof(lst) / sys.getsizeof(gen):,.0f}x")

# === เมื่อไหร่ควรใช้อะไร ===
print("\n=== เมื่อไหร่ควรใช้อะไร ===")
guidelines = [
    ("Generator", "ข้อมูลขนาดใหญ่มาก (GB+)", "ประหยัด RAM"),
    ("Generator", "ต้องการ lazy evaluation", "คำนวณเมื่อต้องการจริงๆ"),
    ("Generator", "Infinite sequences", "สร้างข้อมูลไม่สิ้นสุด"),
    ("Generator", "Pipeline processing", "ส่งข้อมูลผ่าน stages"),
    ("List", "ต้องการ random access", "lst[i], lst[-1]"),
    ("List", "ต้องการใช้ซ้ำหลายรอบ", "loop หลายครั้ง"),
    ("List", "ต้องการ len()", "ทราบจำนวนก่อน"),
    ("List", "ข้อมูลขนาดเล็ก", "ง่ายต่อการ debug"),
]

for approach, use_case, reason in guidelines:
    print(f"  {'✅' if approach == 'Generator' else '📋'} ใช้ {approach}: {use_case}")
    print(f"     เหตุผล: {reason}")
```

---

## สรุป Part 019

✅ **Generator** คือฟังก์ชันพิเศษที่ใช้ `yield` แทน `return` สร้างค่าทีละตัว  
✅ **yield** หยุดฟังก์ชันชั่วคราวและส่งค่ากลับ ครั้งต่อไปที่เรียกจะทำงานต่อจากที่ค้างไว้  
✅ **Generator Expression** `(x for x in ...)` ประหยัด RAM กว่า List Comprehension มาก  
✅ **next()** ดึงค่าถัดไป, **iter()** แปลง iterable เป็น iterator  
✅ **send()** ส่งค่าเข้า generator ทำให้เป็น coroutine ได้  
✅ **Infinite Generator** ต้องมีเงื่อนไขหยุดจากภายนอก  
✅ **yield from** delegate ให้ sub-generator ทำงาน สั้นและอ่านง่ายกว่า  
✅ **itertools** มี utilities ที่มีประสิทธิภาพสูง: chain, cycle, count, islice, product, combinations, permutations  
✅ **Pipeline** เชื่อม generators เข้าด้วยกันเพื่อประมวลผลข้อมูลแบบ lazy  

---

## ➡️ ถัดไป: Part 020 - Functional Programming

*Part 019/100+ | Python Course - Beginner to World-Class*
