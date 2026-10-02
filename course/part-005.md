# Part 005: ลูป for และ while (Loops)
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- ใช้ for loop และ range() ได้อย่างคล่องแคล่ว
- ใช้ while loop สำหรับการวนซ้ำที่ไม่รู้จำนวนรอบ
- ใช้ break, continue, pass ได้
- เข้าใจ else ใน loops
- สร้าง Comprehensions (list, dict, set, generator)

---

## 1. for Loop

### 1.1 พื้นฐาน

```python
# วนซ้ำตาม sequence
fruits = ["apple", "banana", "cherry", "date"]

for fruit in fruits:
    print(fruit)

# วนซ้ำตาม string
for char in "Python":
    print(char, end=" ")  # P y t h o n
print()  # newline

# วนซ้ำตาม range
for i in range(5):
    print(i, end=" ")  # 0 1 2 3 4
print()

# วนซ้ำตาม tuple
for day in ("Mon", "Tue", "Wed", "Thu", "Fri"):
    print(day, end=" ")
print()

# วนซ้ำตาม dict
person = {"name": "Alice", "age": 25, "city": "Bangkok"}

for key in person:              # วน keys
    print(key)

for value in person.values():   # วน values
    print(value)

for key, value in person.items():  # วน key-value pairs
    print(f"{key}: {value}")
```

### 1.2 range() Function

```python
# range(stop) - 0 ถึง stop-1
for i in range(5):
    print(i, end=" ")   # 0 1 2 3 4
print()

# range(start, stop) - start ถึง stop-1
for i in range(1, 6):
    print(i, end=" ")   # 1 2 3 4 5
print()

# range(start, stop, step) - มี step
for i in range(0, 10, 2):
    print(i, end=" ")   # 0 2 4 6 8
print()

# นับถอยหลัง
for i in range(10, 0, -1):
    print(i, end=" ")   # 10 9 8 7 6 5 4 3 2 1
print()

# range เป็น lazy (ไม่สร้าง list จริงๆ)
large_range = range(1_000_000)  # ไม่ใช้ memory มาก
print(type(large_range))        # <class 'range'>
print(large_range[500])         # 500 (เข้าถึงได้)
print(len(large_range))         # 1000000

# แปลงเป็น list
print(list(range(5)))           # [0, 1, 2, 3, 4]
print(list(range(1, 10, 2)))    # [1, 3, 5, 7, 9]
print(list(range(10, 0, -3)))   # [10, 7, 4, 1]
```

### 1.3 enumerate() - ได้ index พร้อมกัน

```python
fruits = ["apple", "banana", "cherry"]

# แบบไม่ใช้ enumerate
for i in range(len(fruits)):
    print(f"{i}: {fruits[i]}")

# ✅ แบบใช้ enumerate (ดีกว่า)
for i, fruit in enumerate(fruits):
    print(f"{i}: {fruit}")

# กำหนด start index
for i, fruit in enumerate(fruits, start=1):
    print(f"{i}. {fruit}")

# Unpack nested
students = [("Alice", 90), ("Bob", 85), ("Charlie", 78)]
for i, (name, score) in enumerate(students, 1):
    print(f"{i}. {name}: {score}")
```

### 1.4 zip() - วนหลาย sequence พร้อมกัน

```python
names = ["Alice", "Bob", "Charlie"]
ages = [25, 30, 35]
cities = ["Bangkok", "Chiang Mai", "Phuket"]

# zip combines sequences
for name, age in zip(names, ages):
    print(f"{name}: {age} ปี")

# zip หลาย sequences
for name, age, city in zip(names, ages, cities):
    print(f"{name}, {age}, {city}")

# zip หยุดเมื่อสั้นที่สุดจบ
long = [1, 2, 3, 4, 5]
short = ["a", "b", "c"]
for n, s in zip(long, short):
    print(n, s)  # 1 a, 2 b, 3 c (หยุดที่ short)

# zip_longest - ใช้ fillvalue สำหรับส่วนที่ขาด
from itertools import zip_longest
for n, s in zip_longest(long, short, fillvalue="?"):
    print(n, s)  # 1 a, 2 b, 3 c, 4 ?, 5 ?

# สร้าง dict จาก zip
person = dict(zip(["name", "age", "city"], ["Alice", 25, "Bangkok"]))
print(person)  # {'name': 'Alice', 'age': 25, 'city': 'Bangkok'}
```

### 1.5 Nested for Loop

```python
# Loop ซ้อน loop
for i in range(1, 4):
    for j in range(1, 4):
        print(f"({i},{j})", end=" ")
    print()

# สร้างตาราง multiplication
print("\nตาราง คูณ 1-5")
print("    ", end="")
for i in range(1, 6):
    print(f"{i:3}", end="")
print()
print("-" * 20)

for i in range(1, 6):
    print(f"{i:3}|", end="")
    for j in range(1, 6):
        print(f"{i*j:3}", end="")
    print()

# Pattern ด้วย nested loop
def print_pattern(n):
    """พิมพ์รูปสามเหลี่ยม"""
    for i in range(1, n+1):
        print("*" * i)

def print_diamond(n):
    """พิมพ์รูปเพชร"""
    for i in range(1, n+1):
        print(" " * (n-i) + "*" * (2*i-1))
    for i in range(n-1, 0, -1):
        print(" " * (n-i) + "*" * (2*i-1))

print("สามเหลี่ยม:")
print_pattern(5)

print("\nเพชร:")
print_diamond(4)
```

---

## 2. while Loop

### 2.1 พื้นฐาน

```python
# while วนซ้ำตราบเท่าที่ condition เป็น True
count = 0

while count < 5:
    print(count, end=" ")  # 0 1 2 3 4
    count += 1
print()

# Infinite loop (ต้อง break ออก!)
import time
attempts = 0
max_attempts = 3

while True:
    attempts += 1
    print(f"พยายามที่ {attempts}")
    
    if attempts >= max_attempts:
        print("หมดจำนวนครั้งแล้ว")
        break

# do-while simulation
# Python ไม่มี do-while แต่ทำได้ด้วย:
while True:
    user_input = input("กรอก 'quit' เพื่อหยุด: ")
    if user_input.lower() == 'quit':
        break
    print(f"คุณกรอก: {user_input}")
```

### 2.2 while กับ Sentinel Value

```python
# รับ input จนกว่าจะกรอก sentinel
numbers = []
print("กรอกตัวเลข (กรอก 0 เพื่อหยุด):")

while True:
    num = float(input("ตัวเลข: "))
    if num == 0:
        break
    numbers.append(num)

if numbers:
    print(f"ตัวเลขที่กรอก: {numbers}")
    print(f"ผลรวม: {sum(numbers)}")
    print(f"เฉลี่ย: {sum(numbers)/len(numbers):.2f}")
    print(f"สูงสุด: {max(numbers)}")
    print(f"ต่ำสุด: {min(numbers)}")
else:
    print("ไม่มีข้อมูล")
```

### 2.3 while สำหรับ Retry Logic

```python
import random
import time

def unstable_operation():
    """ฟังก์ชันที่อาจ fail"""
    return random.random() > 0.6  # 40% chance of success

def retry_with_backoff(func, max_retries=5):
    """ลองใหม่ด้วย exponential backoff"""
    retry_count = 0
    
    while retry_count < max_retries:
        if func():
            print(f"สำเร็จหลังจากลอง {retry_count + 1} ครั้ง")
            return True
        
        retry_count += 1
        if retry_count < max_retries:
            wait_time = 2 ** retry_count  # 2, 4, 8, 16 วินาที
            print(f"ล้มเหลวครั้งที่ {retry_count}, รอ {wait_time} วินาที...")
            # time.sleep(wait_time)  # ปิด comment ใน production
    
    print("ล้มเหลวทั้งหมด")
    return False

retry_with_backoff(unstable_operation)
```

---

## 3. break, continue, pass

### 3.1 break - หยุด loop

```python
# หยุด loop เมื่อพบเงื่อนไข
numbers = [1, 4, 7, 2, 9, 3, 8, 5]

# หาตัวเลขแรกที่มากกว่า 6
for num in numbers:
    if num > 6:
        print(f"พบ {num}")
        break  # หยุด loop ทันที
else:
    print("ไม่พบ")  # else ของ for (จะทำงานถ้า loop จบปกติ ไม่ถูก break)

# ใช้ break ใน nested loop
found = False
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
target = 5

for row_idx, row in enumerate(matrix):
    for col_idx, value in enumerate(row):
        if value == target:
            print(f"พบ {target} ที่ ({row_idx}, {col_idx})")
            found = True
            break  # หยุดแค่ inner loop
    if found:
        break  # หยุด outer loop ด้วย
```

### 3.2 continue - ข้ามรอบนี้

```python
# ข้ามรอบที่ไม่ต้องการ
for i in range(10):
    if i % 2 == 0:      # ถ้าเลขคู่
        continue         # ข้ามไปรอบถัดไป
    print(i, end=" ")   # 1 3 5 7 9
print()

# กรอง negative numbers
numbers = [3, -1, 4, -2, 5, -3, 2]
positives = []

for num in numbers:
    if num < 0:
        continue  # ข้ามลบ
    positives.append(num)

print(f"จำนวนบวก: {positives}")   # [3, 4, 5, 2]

# Process lines จาก file (จำลอง)
lines = [
    "# comment line",
    "name: Alice",
    "",  # empty line
    "# another comment",
    "age: 25",
    "  ",  # whitespace line
    "city: Bangkok",
]

data = {}
for line in lines:
    line = line.strip()
    
    if not line:          # skip empty lines
        continue
    if line.startswith("#"):  # skip comments
        continue
    
    key, _, value = line.partition(":")
    data[key.strip()] = value.strip()

print(data)  # {'name': 'Alice', 'age': '25', 'city': 'Bangkok'}
```

### 3.3 pass - ไม่ทำอะไร

```python
# pass เป็น placeholder สำหรับโค้ดที่ยังไม่ได้เขียน
def todo_function():
    pass  # TODO: implement this

class EmptyClass:
    pass

# ใช้ pass ใน loop เพื่อ placeholder
for i in range(5):
    pass  # จะเพิ่มโค้ดทีหลัง

# ใช้ pass เพื่อ ignore exception
try:
    risky_operation = 1 / 0
except ZeroDivisionError:
    pass  # ignore error intentionally

# ⚠️ ไม่แนะนำใช้ pass บ่อยในโค้ด production
# ควรมีเหตุผลชัดเจน
```

---

## 4. else ใน Loops

Python มี else clause ใน for และ while ซึ่งทำงานเมื่อ loop จบปกติ (ไม่ถูก break)

```python
# for-else
def find_prime(n):
    """ตรวจสอบว่า n เป็นเลขเฉพาะหรือไม่"""
    if n < 2:
        return False
    
    for i in range(2, int(n**0.5) + 1):
        if n % i == 0:
            break  # พบตัวหาร = ไม่ใช่เลขเฉพาะ
    else:
        return True  # ไม่ถูก break = เลขเฉพาะ
    
    return False

for num in range(1, 20):
    if find_prime(num):
        print(num, end=" ")
print()  # 2 3 5 7 11 13 17 19

# while-else
def search_in_list(lst, target):
    """ค้นหาใน list"""
    i = 0
    while i < len(lst):
        if lst[i] == target:
            print(f"พบ {target} ที่ index {i}")
            break
        i += 1
    else:
        print(f"ไม่พบ {target}")

search_in_list([1, 5, 3, 8, 2], 8)   # พบ 8 ที่ index 3
search_in_list([1, 5, 3, 8, 2], 10)  # ไม่พบ 10
```

---

## 5. Comprehensions

### 5.1 List Comprehension

```python
# สร้าง list จาก expression
# รูปแบบ: [expression for item in iterable if condition]

# แบบปกติ
squares = []
for i in range(10):
    squares.append(i ** 2)

# ✅ List comprehension (กระชับกว่า)
squares = [i ** 2 for i in range(10)]
print(squares)  # [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]

# มี condition
even_squares = [i ** 2 for i in range(10) if i % 2 == 0]
print(even_squares)  # [0, 4, 16, 36, 64]

# Transformation
names = ["alice", "bob", "charlie", "diana"]
upper_names = [name.upper() for name in names]
print(upper_names)   # ['ALICE', 'BOB', 'CHARLIE', 'DIANA']

# Filtering
long_names = [name for name in names if len(name) > 4]
print(long_names)    # ['alice', 'charlie', 'diana']

# Nested list comprehension
matrix = [[j for j in range(5)] for i in range(3)]
print(matrix)
# [[0, 1, 2, 3, 4], [0, 1, 2, 3, 4], [0, 1, 2, 3, 4]]

# Flatten nested list
nested = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
flat = [num for row in nested for num in row]
print(flat)  # [1, 2, 3, 4, 5, 6, 7, 8, 9]

# Multiple conditions
result = [x for x in range(100) if x % 2 == 0 if x % 3 == 0]
print(result)  # [0, 6, 12, 18, ...] (หาร 2 และ 3 ลงตัว)

# Conditional expression ใน comprehension
labels = ["pass" if score >= 60 else "fail" for score in [85, 72, 55, 91, 48]]
print(labels)  # ['pass', 'pass', 'fail', 'pass', 'fail']
```

### 5.2 Dict Comprehension

```python
# สร้าง dict จาก expression
# รูปแบบ: {key: value for item in iterable if condition}

# แปลง list เป็น dict
fruits = ["apple", "banana", "cherry"]
fruit_lengths = {fruit: len(fruit) for fruit in fruits}
print(fruit_lengths)  # {'apple': 5, 'banana': 6, 'cherry': 6}

# Invert dictionary
original = {"a": 1, "b": 2, "c": 3}
inverted = {v: k for k, v in original.items()}
print(inverted)  # {1: 'a', 2: 'b', 3: 'c'}

# กรองและแปลง
scores = {"Alice": 85, "Bob": 72, "Charlie": 91, "Diana": 55}
passed = {name: score for name, score in scores.items() if score >= 70}
print(passed)  # {'Alice': 85, 'Bob': 72, 'Charlie': 91}

# Square dictionary
square_dict = {x: x**2 for x in range(1, 6)}
print(square_dict)  # {1: 1, 2: 4, 3: 9, 4: 16, 5: 25}

# Conditional value
grade_dict = {
    name: "Pass" if score >= 70 else "Fail"
    for name, score in scores.items()
}
print(grade_dict)

# สร้าง lookup dictionary
words = ["apple", "application", "apply", "banana", "band"]
first_letter_dict = {}
for word in words:
    letter = word[0]
    if letter not in first_letter_dict:
        first_letter_dict[letter] = []
    first_letter_dict[letter].append(word)

# ✅ ใช้ defaultdict แทน
from collections import defaultdict
first_letter = defaultdict(list)
for word in words:
    first_letter[word[0]].append(word)
print(dict(first_letter))
```

### 5.3 Set Comprehension

```python
# สร้าง set จาก expression
# รูปแบบ: {expression for item in iterable if condition}

# สร้าง set ของ squares
squares_set = {x**2 for x in range(-5, 6)}
print(squares_set)  # {0, 1, 4, 9, 16, 25} (ไม่ซ้ำ, ไม่เรียง)

# กรองตัวอักษรไม่ซ้ำ
unique_chars = {char.lower() for char in "Hello World" if char != " "}
print(sorted(unique_chars))  # ['d', 'e', 'h', 'l', 'o', 'r', 'w']

# หาคำที่ไม่ซ้ำ
text = "the cat sat on the mat"
unique_words = {word for word in text.split()}
print(unique_words)
```

### 5.4 Generator Expression

```python
# Generator ใช้ () แทน []
# ไม่สร้างข้อมูลทั้งหมดล่วงหน้า (lazy evaluation)

# List comprehension - สร้างทันที
squares_list = [x**2 for x in range(10)]  # ใช้ memory มากกว่า

# Generator expression - สร้างทีละตัวเมื่อต้องการ
squares_gen = (x**2 for x in range(10))   # ใช้ memory น้อยกว่า
print(type(squares_gen))                   # <class 'generator'>

# วิธีใช้ generator
for sq in squares_gen:
    print(sq, end=" ")
print()

# หรือแปลงเป็น list
squares_gen = (x**2 for x in range(10))
print(list(squares_gen))

# sum ไม่ต้องสร้าง list ก่อน
total = sum(x**2 for x in range(1000000))
print(f"Sum of squares: {total}")

# any() และ all() กับ generator (short-circuit)
numbers = [2, 4, 6, 8, 10]
print(all(x % 2 == 0 for x in numbers))  # True (ทุกตัวเป็นเลขคู่)
print(any(x > 9 for x in numbers))       # True (มีอย่างน้อยหนึ่งตัว > 9)

# เปรียบเทียบ memory
import sys
list_comp = [x**2 for x in range(1000)]
gen_exp = (x**2 for x in range(1000))
print(f"List: {sys.getsizeof(list_comp)} bytes")  # ~9000 bytes
print(f"Generator: {sys.getsizeof(gen_exp)} bytes")  # ~120 bytes
```

---

## 6. itertools - เครื่องมือสำหรับ iteration

```python
import itertools

# count - นับไม่หยุด
counter = itertools.count(10, 2)  # เริ่ม 10, step 2
for _ in range(5):
    print(next(counter), end=" ")  # 10 12 14 16 18
print()

# cycle - วนซ้ำ
colors = itertools.cycle(["red", "green", "blue"])
for _ in range(7):
    print(next(colors), end=" ")  # red green blue red green blue red
print()

# repeat - ทำซ้ำ n ครั้ง
for x in itertools.repeat("Hello", 3):
    print(x)  # Hello Hello Hello

# chain - ต่อ iterables
for item in itertools.chain([1, 2, 3], [4, 5, 6], [7, 8, 9]):
    print(item, end=" ")  # 1 2 3 4 5 6 7 8 9
print()

# combinations
for combo in itertools.combinations("ABC", 2):
    print(''.join(combo), end=" ")  # AB AC BC
print()

# permutations
for perm in itertools.permutations("ABC", 2):
    print(''.join(perm), end=" ")  # AB AC BA BC CA CB
print()

# product (Cartesian product)
for pair in itertools.product([1, 2], ["a", "b"]):
    print(pair, end=" ")  # (1,'a') (1,'b') (2,'a') (2,'b')
print()

# groupby
data = [
    {"name": "Alice", "dept": "IT"},
    {"name": "Bob", "dept": "HR"},
    {"name": "Charlie", "dept": "IT"},
    {"name": "Diana", "dept": "HR"},
    {"name": "Eve", "dept": "IT"},
]

# ต้อง sort ก่อน groupby
data.sort(key=lambda x: x["dept"])
for dept, group in itertools.groupby(data, key=lambda x: x["dept"]):
    print(f"\n{dept}:")
    for person in group:
        print(f"  {person['name']}")

# islice - slice generator
big_data = (x**2 for x in range(1_000_000))
first_ten = list(itertools.islice(big_data, 10))
print(first_ten)  # [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
```

---

## 7. ตัวอย่างโปรแกรม: Data Processing

```python
# data_processor.py
"""
ตัวอย่างการใช้ loops ในการ process ข้อมูล
"""

# ข้อมูลนักเรียน
students_data = [
    {"name": "Alice", "scores": [85, 90, 78, 92, 88]},
    {"name": "Bob", "scores": [72, 68, 75, 80, 71]},
    {"name": "Charlie", "scores": [91, 95, 88, 93, 96]},
    {"name": "Diana", "scores": [55, 62, 58, 70, 65]},
    {"name": "Eve", "scores": [88, 92, 95, 89, 91]},
]

def calculate_statistics(scores):
    """คำนวณสถิติจาก scores"""
    n = len(scores)
    total = sum(scores)
    average = total / n
    minimum = min(scores)
    maximum = max(scores)
    
    # คำนวณ standard deviation
    variance = sum((x - average) ** 2 for x in scores) / n
    std_dev = variance ** 0.5
    
    return {
        "total": total,
        "average": average,
        "min": minimum,
        "max": maximum,
        "std_dev": std_dev,
    }

def get_grade(average):
    if average >= 90:
        return "A"
    elif average >= 80:
        return "B"
    elif average >= 70:
        return "C"
    elif average >= 60:
        return "D"
    else:
        return "F"

# Process ข้อมูล
processed = []
for student in students_data:
    stats = calculate_statistics(student["scores"])
    grade = get_grade(stats["average"])
    processed.append({
        **student,
        **stats,
        "grade": grade,
    })

# เรียงตาม average (สูงไปต่ำ)
processed.sort(key=lambda s: s["average"], reverse=True)

# แสดงผล
print("=" * 70)
print(f"{'อันดับ':<6} {'ชื่อ':<12} {'เฉลี่ย':<8} {'สูงสุด':<8} {'ต่ำสุด':<8} เกรด")
print("=" * 70)

for rank, student in enumerate(processed, 1):
    print(f"{rank:<6} {student['name']:<12} "
          f"{student['average']:<8.1f} "
          f"{student['max']:<8} "
          f"{student['min']:<8} "
          f"{student['grade']}")

print("=" * 70)

# สรุปห้องเรียน
all_averages = [s["average"] for s in processed]
class_average = sum(all_averages) / len(all_averages)
grade_distribution = {}

for student in processed:
    grade = student["grade"]
    grade_distribution[grade] = grade_distribution.get(grade, 0) + 1

print(f"\nเฉลี่ยห้อง: {class_average:.1f}")
print("การกระจายเกรด:")
for grade in sorted(grade_distribution.keys()):
    count = grade_distribution[grade]
    bar = "█" * count
    print(f"  {grade}: {bar} ({count} คน)")

# Top performers
print("\nนักเรียนที่ได้เกรด A:")
top_students = [s["name"] for s in processed if s["grade"] == "A"]
for name in top_students:
    print(f"  ⭐ {name}")
```

---

## 8. ตัวอย่าง: Word Frequency Counter

```python
# word_frequency.py

def count_words(text):
    """นับความถี่ของคำในข้อความ"""
    # แปลงเป็น lowercase และแยกคำ
    words = text.lower().split()
    
    # ลบ punctuation
    cleaned_words = []
    for word in words:
        clean = ''.join(c for c in word if c.isalnum())
        if clean:
            cleaned_words.append(clean)
    
    # นับความถี่
    frequency = {}
    for word in cleaned_words:
        frequency[word] = frequency.get(word, 0) + 1
    
    return frequency

def display_top_words(frequency, top_n=10):
    """แสดงคำที่ใช้บ่อยที่สุด"""
    # เรียงตามความถี่
    sorted_words = sorted(frequency.items(), key=lambda x: x[1], reverse=True)
    
    print(f"\n=== Top {top_n} คำที่ใช้บ่อย ===")
    max_count = sorted_words[0][1] if sorted_words else 0
    
    for i, (word, count) in enumerate(sorted_words[:top_n], 1):
        bar_length = int(count / max_count * 30)
        bar = "█" * bar_length
        print(f"{i:3}. {word:<15} {count:5} {bar}")

# ข้อความตัวอย่าง
sample_text = """
Python is a high-level, general-purpose programming language.
Python's design philosophy emphasizes code readability.
Python is dynamically typed and garbage-collected.
Python supports multiple programming paradigms.
Python is often described as batteries included because 
Python comes with a comprehensive standard library.
The Python community is large and active.
Python programmers are called Pythonistas.
"""

frequency = count_words(sample_text)
display_top_words(frequency, 10)

print(f"\nจำนวนคำทั้งหมด: {sum(frequency.values())}")
print(f"คำไม่ซ้ำ: {len(frequency)}")
```

---

## 9. Exercises

### Exercise 1: FizzBuzz
```python
# FizzBuzz Classic
# - ถ้าหาร 3 ลงตัว: Fizz
# - ถ้าหาร 5 ลงตัว: Buzz
# - ถ้าหาร 15 ลงตัว: FizzBuzz
# - อื่นๆ: แสดงตัวเลข

for i in range(1, 101):
    if i % 15 == 0:
        print("FizzBuzz")
    elif i % 3 == 0:
        print("Fizz")
    elif i % 5 == 0:
        print("Buzz")
    else:
        print(i)
```

### Exercise 2: Pattern Printing
```python
def print_patterns():
    """พิมพ์ patterns ต่างๆ"""
    n = 5
    
    # 1. Number Triangle
    print("Number Triangle:")
    for i in range(1, n+1):
        for j in range(1, i+1):
            print(j, end="")
        print()
    
    print()
    
    # 2. Reversed Triangle
    print("Reversed Triangle:")
    for i in range(n, 0, -1):
        print("*" * i)
    
    print()
    
    # 3. Pyramid
    print("Pyramid:")
    for i in range(1, n+1):
        spaces = " " * (n - i)
        stars = "*" * (2*i - 1)
        print(spaces + stars)
    
    print()
    
    # 4. Hollow Square
    print("Hollow Square:")
    for i in range(n):
        for j in range(n):
            if i == 0 or i == n-1 or j == 0 or j == n-1:
                print("*", end="")
            else:
                print(" ", end="")
        print()

print_patterns()
```

### Exercise 3: Simple Menu System
```python
def menu_system():
    """ระบบเมนูง่ายๆ"""
    menu = {
        "1": ("เพิ่มนักเรียน", add_student),
        "2": ("ดูรายชื่อ", show_students),
        "3": ("ค้นหา", search_student),
        "4": ("ออก", None),
    }
    
    students = []
    
    def add_student():
        name = input("ชื่อนักเรียน: ")
        score = float(input("คะแนน: "))
        students.append({"name": name, "score": score})
        print(f"เพิ่ม {name} สำเร็จ")
    
    def show_students():
        if not students:
            print("ไม่มีข้อมูล")
            return
        print("\nรายชื่อนักเรียน:")
        for i, s in enumerate(students, 1):
            print(f"  {i}. {s['name']}: {s['score']}")
    
    def search_student():
        name = input("ค้นหาชื่อ: ")
        found = [s for s in students if name.lower() in s['name'].lower()]
        if found:
            for s in found:
                print(f"  {s['name']}: {s['score']}")
        else:
            print("ไม่พบ")
    
    while True:
        print("\n=== เมนู ===")
        for key, (label, _) in menu.items():
            print(f"  {key}. {label}")
        
        choice = input("\nเลือก: ").strip()
        
        if choice not in menu:
            print("ตัวเลือกไม่ถูกต้อง")
            continue
        
        label, action = menu[choice]
        
        if choice == "4":
            print("ลาก่อน!")
            break
        
        action()

# menu_system()  # ยกเว้น comment เพื่อรัน
```

---

## 10. สรุป Part 005

### สิ่งที่เรียนรู้:

✅ **for loop** - วนซ้ำตาม sequence  
✅ **range()** - สร้าง sequence ตัวเลข  
✅ **enumerate()** - ได้ index พร้อมกัน  
✅ **zip()** - วนหลาย sequence พร้อมกัน  
✅ **while loop** - วนซ้ำตาม condition  
✅ **break/continue/pass** - ควบคุม loop  
✅ **else ใน loops** - ทำงานเมื่อ loop จบปกติ  
✅ **List/Dict/Set Comprehension** - สร้างข้อมูลกระชับ  
✅ **Generator Expression** - lazy evaluation  
✅ **itertools** - เครื่องมือ iteration ขั้นสูง  

### Quick Reference:

```python
# for loop
for item in iterable:
    pass

for i, item in enumerate(iterable):
    pass

for a, b in zip(list1, list2):
    pass

# while loop
while condition:
    pass

while True:
    if done:
        break

# Comprehensions
squares = [x**2 for x in range(10)]
evens = [x for x in range(10) if x % 2 == 0]
sq_dict = {x: x**2 for x in range(10)}
sq_set = {x**2 for x in range(10)}
sq_gen = (x**2 for x in range(10))  # lazy
```

---

## ➡️ ถัดไป: Part 006 - ฟังก์ชัน (Functions)

ใน Part ถัดไป เราจะเรียนรู้:
- การสร้างฟังก์ชัน
- Parameters และ Arguments ทุกรูปแบบ
- Return values
- Scope ของตัวแปร
- Docstrings
- Lambda functions
- Higher-order functions

---

*Part 005/100+ | Python Course - Beginner to World-Class*
