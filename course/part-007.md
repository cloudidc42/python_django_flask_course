# Part 007: Lists และ List Methods
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- สร้างและใช้งาน List ได้อย่างคล่องแคล่ว
- ใช้ List Methods ได้ครบทุก method
- ทำ Slicing ขั้นสูงได้
- เรียงลำดับข้อมูลด้วยวิธีต่างๆ
- ใช้ Nested Lists ได้
- เขียน List Comprehension ขั้นสูงได้

---

## 1. List พื้นฐาน

```python
# สร้าง List ด้วยวิธีต่างๆ
empty_list = []                          # list ว่าง
numbers = [1, 2, 3, 4, 5]              # list ตัวเลข
fruits = ["apple", "banana", "cherry"]  # list string
mixed = [1, "hello", 3.14, True, None]  # list ผสม type

# สร้างด้วย list() constructor
from_range = list(range(1, 11))         # [1, 2, 3, ..., 10]
from_string = list("Python")            # ['P', 'y', 't', 'h', 'o', 'n']
from_tuple = list((1, 2, 3))           # [1, 2, 3]

print(numbers)       # [1, 2, 3, 4, 5]
print(from_range)    # [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
print(from_string)   # ['P', 'y', 't', 'h', 'o', 'n']

# การเข้าถึงสมาชิก (Indexing)
fruits = ["apple", "banana", "cherry", "date", "elderberry"]

print(fruits[0])    # apple (index แรก)
print(fruits[2])    # cherry
print(fruits[-1])   # elderberry (index สุดท้าย)
print(fruits[-2])   # date (นับจากท้าย)

# len() - ความยาวของ list
print(len(fruits))  # 5

# ตรวจสอบว่ามีสมาชิกหรือไม่ (Membership Test)
print("apple" in fruits)     # True
print("mango" in fruits)     # False
print("mango" not in fruits) # True
```

---

## 2. List Methods - ครบทุก Method

### 2.1 append() - เพิ่มสมาชิกท้าย list

```python
fruits = ["apple", "banana"]

# append() เพิ่มสมาชิก 1 รายการท้าย list
fruits.append("cherry")
print(fruits)  # ['apple', 'banana', 'cherry']

fruits.append("date")
fruits.append("elderberry")
print(fruits)  # ['apple', 'banana', 'cherry', 'date', 'elderberry']

# append รับค่าได้ทุก type รวมถึง list
fruits.append([1, 2, 3])  # เพิ่ม list เป็นสมาชิก 1 ตัว
print(fruits)  # ['apple', 'banana', 'cherry', 'date', 'elderberry', [1, 2, 3]]
print(len(fruits))  # 6

# ตัวอย่างการใช้จริง: รวบรวมข้อมูล
scores = []
for i in range(5):
    scores.append(i * 10)
print(scores)  # [0, 10, 20, 30, 40]
```

### 2.2 extend() - เพิ่มสมาชิกหลายรายการ

```python
list1 = [1, 2, 3]
list2 = [4, 5, 6]

# extend() รับ iterable และเพิ่มทีละตัว (ต่างจาก append)
list1.extend(list2)
print(list1)  # [1, 2, 3, 4, 5, 6]

# extend กับ string
letters = ['a', 'b', 'c']
letters.extend("xyz")
print(letters)  # ['a', 'b', 'c', 'x', 'y', 'z']

# extend กับ range
nums = [1, 2, 3]
nums.extend(range(4, 8))
print(nums)  # [1, 2, 3, 4, 5, 6, 7]

# เปรียบเทียบ append vs extend
a = [1, 2, 3]
b = [1, 2, 3]

a.append([4, 5])   # เพิ่ม list เป็น 1 สมาชิก
b.extend([4, 5])   # เพิ่ม 4, 5 เป็น 2 สมาชิก

print(a)  # [1, 2, 3, [4, 5]]   ← มี nested list
print(b)  # [1, 2, 3, 4, 5]     ← flat list

# + operator ก็ได้ผลคล้ายกัน (แต่สร้าง list ใหม่)
c = [1, 2, 3] + [4, 5, 6]
print(c)  # [1, 2, 3, 4, 5, 6]
```

### 2.3 insert() - แทรกสมาชิกที่ตำแหน่งที่กำหนด

```python
fruits = ["apple", "cherry", "elderberry"]

# insert(index, value) - แทรก value ที่ตำแหน่ง index
fruits.insert(1, "banana")          # แทรก "banana" ที่ index 1
print(fruits)  # ['apple', 'banana', 'cherry', 'elderberry']

fruits.insert(3, "date")            # แทรก "date" ที่ index 3
print(fruits)  # ['apple', 'banana', 'cherry', 'date', 'elderberry']

# แทรกที่ต้น list
fruits.insert(0, "avocado")
print(fruits)  # ['avocado', 'apple', 'banana', 'cherry', 'date', 'elderberry']

# index เกินขนาดจะแทรกท้ายสุด
fruits.insert(100, "fig")
print(fruits[-1])  # fig

# index ลบ - นับจากท้าย
nums = [1, 2, 3, 4, 5]
nums.insert(-1, 99)  # แทรกก่อน index -1 (ก่อนสมาชิกสุดท้าย)
print(nums)  # [1, 2, 3, 4, 99, 5]
```

### 2.4 remove() - ลบสมาชิกตามค่า

```python
fruits = ["apple", "banana", "cherry", "banana", "date"]

# remove() ลบสมาชิกแรกที่พบ
fruits.remove("banana")
print(fruits)  # ['apple', 'cherry', 'banana', 'date']

# ลบอีกครั้ง
fruits.remove("banana")
print(fruits)  # ['apple', 'cherry', 'date']

# ถ้าไม่พบ - ได้ ValueError
try:
    fruits.remove("mango")
except ValueError as e:
    print(f"Error: {e}")  # Error: list.remove(x): x not in list

# ตรวจสอบก่อนลบ
if "apple" in fruits:
    fruits.remove("apple")
    print("ลบ apple แล้ว")
print(fruits)  # ['cherry', 'date']
```

### 2.5 pop() - ลบและคืนค่าสมาชิก

```python
fruits = ["apple", "banana", "cherry", "date"]

# pop() ไม่มี argument - ลบและคืนสมาชิกสุดท้าย
last = fruits.pop()
print(f"ลบ: {last}")    # ลบ: date
print(fruits)            # ['apple', 'banana', 'cherry']

# pop(index) - ลบและคืนสมาชิกที่ index
second = fruits.pop(1)
print(f"ลบ: {second}")  # ลบ: banana
print(fruits)            # ['apple', 'cherry']

# Stack (LIFO) โดยใช้ append + pop
stack = []
stack.append("first")
stack.append("second")
stack.append("third")
print(stack)         # ['first', 'second', 'third']
print(stack.pop())   # third  ← ออกทีหลัง
print(stack.pop())   # second
print(stack.pop())   # first  ← ออกก่อน

# Queue โดยใช้ pop(0) (แต่ใช้ deque ดีกว่า)
queue = []
queue.append("first")
queue.append("second")
queue.append("third")
print(queue.pop(0))  # first  ← ออกก่อน
print(queue.pop(0))  # second
```

### 2.6 sort() - เรียงลำดับ (in-place)

```python
# sort() แก้ไข list โดยตรง (in-place)
numbers = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3]

numbers.sort()
print(numbers)  # [1, 1, 2, 3, 3, 4, 5, 5, 6, 9]

# เรียงจากมากไปน้อย
numbers.sort(reverse=True)
print(numbers)  # [9, 6, 5, 5, 4, 3, 3, 2, 1, 1]

# เรียง string
fruits = ["cherry", "apple", "banana", "date"]
fruits.sort()
print(fruits)  # ['apple', 'banana', 'cherry', 'date']

fruits.sort(reverse=True)
print(fruits)  # ['date', 'cherry', 'banana', 'apple']

# sort ด้วย key function
words = ["banana", "Apple", "cherry", "Date", "elderberry"]
words.sort()                        # เรียงตาม ASCII (uppercase ก่อน)
print(words)  # ['Apple', 'Date', 'banana', 'cherry', 'elderberry']

words.sort(key=str.lower)           # case-insensitive
print(words)  # ['Apple', 'banana', 'cherry', 'Date', 'elderberry']

# เรียงตามความยาว
words.sort(key=len)
print(words)  # ['Apple', 'Date', 'banana', 'cherry', 'elderberry']

# เรียง list ของ dict
students = [
    {"name": "Alice", "grade": 85},
    {"name": "Bob", "grade": 92},
    {"name": "Charlie", "grade": 78},
    {"name": "Diana", "grade": 95},
]

students.sort(key=lambda s: s["grade"])
for s in students:
    print(f"{s['name']}: {s['grade']}")
# Charlie: 78
# Alice: 85
# Bob: 92
# Diana: 95

# เรียงจากมากไปน้อย
students.sort(key=lambda s: s["grade"], reverse=True)
print(students[0])  # {'name': 'Diana', 'grade': 95}
```

### 2.7 reverse() - กลับลำดับ list

```python
numbers = [1, 2, 3, 4, 5]

# reverse() แก้ไข list โดยตรง
numbers.reverse()
print(numbers)  # [5, 4, 3, 2, 1]

# หรือใช้ slicing (สร้าง list ใหม่)
original = [1, 2, 3, 4, 5]
reversed_list = original[::-1]
print(original)       # [1, 2, 3, 4, 5]  ← ไม่เปลี่ยน
print(reversed_list)  # [5, 4, 3, 2, 1]  ← list ใหม่

# reversed() - iterator
for item in reversed([1, 2, 3, 4, 5]):
    print(item, end=" ")  # 5 4 3 2 1
print()
```

### 2.8 copy() - คัดลอก list

```python
original = [1, 2, 3, 4, 5]

# copy() - shallow copy
copied = original.copy()
copied.append(6)

print(original)  # [1, 2, 3, 4, 5]  ← ไม่เปลี่ยน
print(copied)    # [1, 2, 3, 4, 5, 6]

# วิธีอื่นในการ copy
copy2 = original[:]          # slicing
copy3 = list(original)       # list constructor
import copy
copy4 = copy.copy(original)  # copy module

# ⚠️ Shallow vs Deep copy
nested = [[1, 2], [3, 4], [5, 6]]

shallow = nested.copy()     # หรือ nested[:]
shallow[0].append(99)       # แก้ nested list ข้างใน

print(nested)   # [[1, 2, 99], [3, 4], [5, 6]]  ← ถูกกระทบ!
print(shallow)  # [[1, 2, 99], [3, 4], [5, 6]]

# Deep copy - แก้ปัญหา
import copy
nested = [[1, 2], [3, 4], [5, 6]]
deep = copy.deepcopy(nested)
deep[0].append(99)

print(nested)  # [[1, 2], [3, 4], [5, 6]]   ← ปลอดภัย
print(deep)    # [[1, 2, 99], [3, 4], [5, 6]]
```

### 2.9 clear() - ลบสมาชิกทั้งหมด

```python
fruits = ["apple", "banana", "cherry"]
print(fruits)  # ['apple', 'banana', 'cherry']

fruits.clear()
print(fruits)  # []
print(len(fruits))  # 0

# ต่างจากการ reassign
fruits2 = ["apple", "banana"]
fruits2 = []     # สร้าง list ใหม่ (ไม่แก้ไข object เดิม)

# clear() แก้ไข object เดิมโดยตรง (สำคัญเมื่อมีหลาย reference)
shared = [1, 2, 3]
ref1 = shared
ref2 = shared

shared.clear()
print(ref1)  # []  ← ถูกกระทบ
print(ref2)  # []  ← ถูกกระทบ
```

### 2.10 count() - นับจำนวน

```python
numbers = [1, 2, 3, 2, 1, 4, 1, 5, 2]

print(numbers.count(1))  # 3
print(numbers.count(2))  # 3
print(numbers.count(4))  # 1
print(numbers.count(9))  # 0

# นับใน string list
fruits = ["apple", "banana", "apple", "cherry", "apple"]
print(fruits.count("apple"))   # 3
print(fruits.count("mango"))   # 0

# ตัวอย่างการใช้: หาค่าที่พบมากที่สุด (mode)
data = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5]
unique_values = set(data)
mode = max(unique_values, key=data.count)
print(f"ค่าที่พบมากที่สุด: {mode}")  # ค่าที่พบมากที่สุด: 5
```

### 2.11 index() - หาตำแหน่ง

```python
fruits = ["apple", "banana", "cherry", "banana", "date"]

# index() คืน index แรกที่พบ
print(fruits.index("banana"))   # 1
print(fruits.index("cherry"))   # 2

# index(value, start) - เริ่มหาจาก index start
print(fruits.index("banana", 2))  # 3  ← พบ banana ตัวที่สองที่ index 3

# index(value, start, end) - หาในช่วง start:end
print(fruits.index("banana", 0, 3))  # 1

# ถ้าไม่พบ - ValueError
try:
    fruits.index("mango")
except ValueError:
    print("ไม่พบ mango")

# ตรวจสอบก่อน
if "cherry" in fruits:
    pos = fruits.index("cherry")
    print(f"cherry อยู่ที่ index {pos}")  # cherry อยู่ที่ index 2
```

---

## 3. Slicing ขั้นสูง

```python
# Syntax: list[start:stop:step]
numbers = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

# พื้นฐาน
print(numbers[2:5])    # [2, 3, 4]     start=2, stop=5 (ไม่รวม)
print(numbers[:4])     # [0, 1, 2, 3]  ตั้งแต่ต้น
print(numbers[6:])     # [6, 7, 8, 9]  ถึงท้าย
print(numbers[:])      # [0,...,9]     ทั้งหมด (copy)

# Step
print(numbers[::2])    # [0, 2, 4, 6, 8]   ทุก 2 ตัว
print(numbers[1::2])   # [1, 3, 5, 7, 9]   เริ่มจาก 1
print(numbers[::3])    # [0, 3, 6, 9]       ทุก 3 ตัว
print(numbers[::-1])   # [9, 8, ..., 0]    กลับหลัง

# Negative index
print(numbers[-3:])    # [7, 8, 9]    3 ตัวสุดท้าย
print(numbers[:-3])    # [0,...,6]    ทุกตัวยกเว้น 3 ตัวสุดท้าย
print(numbers[-5:-2])  # [5, 6, 7]

# ตัวอย่างที่ใช้จริง
data = list(range(20))

# แบ่งเป็น chunks
chunk_size = 5
chunks = [data[i:i+chunk_size] for i in range(0, len(data), chunk_size)]
print(chunks)  # [[0,1,2,3,4], [5,6,7,8,9], [10,11,12,13,14], [15,16,17,18,19]]

# Slicing สำหรับ pagination
def paginate(items, page_size, page_num):
    """คืน items ของ page ที่ระบุ (เริ่มที่ 1)"""
    start = (page_num - 1) * page_size
    end = start + page_size
    return items[start:end]

items = list(range(1, 26))  # 1 ถึง 25
print(paginate(items, 5, 1))  # [1, 2, 3, 4, 5]
print(paginate(items, 5, 3))  # [11, 12, 13, 14, 15]

# Modify slice
numbers = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
numbers[2:5] = [20, 30, 40]
print(numbers)  # [0, 1, 20, 30, 40, 5, 6, 7, 8, 9]

numbers[2:5] = []    # ลบ slice
print(numbers)  # [0, 1, 5, 6, 7, 8, 9]

numbers[1:1] = [10, 11]  # แทรก
print(numbers)  # [0, 10, 11, 1, 5, 6, 7, 8, 9]
```

---

## 4. การเรียงลำดับ

### 4.1 sorted() - สร้าง list ใหม่ที่เรียงแล้ว

```python
# sorted() ต่างจาก sort() ตรงที่ไม่แก้ไข list เดิม
original = [3, 1, 4, 1, 5, 9, 2, 6]

sorted_asc = sorted(original)
sorted_desc = sorted(original, reverse=True)

print(original)    # [3, 1, 4, 1, 5, 9, 2, 6]  ← ไม่เปลี่ยน
print(sorted_asc)  # [1, 1, 2, 3, 4, 5, 6, 9]
print(sorted_desc) # [9, 6, 5, 4, 3, 2, 1, 1]

# sorted() กับ key
words = ["banana", "apple", "Cherry", "date", "Elderberry"]
print(sorted(words))                     # Case-sensitive
print(sorted(words, key=str.lower))      # Case-insensitive
print(sorted(words, key=len))            # เรียงตามความยาว
print(sorted(words, key=len, reverse=True))  # ยาวสุดก่อน

# เรียง tuple
points = [(3, 4), (1, 2), (5, 0), (2, 8), (1, 6)]
print(sorted(points))              # เรียงตาม x แล้วค่อย y
print(sorted(points, key=lambda p: p[1]))  # เรียงตาม y

# เรียงหลายเกณฑ์
from operator import itemgetter

students = [
    ("Alice", 2, 85),
    ("Bob", 1, 90),
    ("Charlie", 2, 90),
    ("Diana", 1, 85),
]
# เรียงตามชั้น (index 1) แล้วตามคะแนน (index 2) จากมากไปน้อย
students_sorted = sorted(students, key=itemgetter(1, 2))
for s in students_sorted:
    print(s)
# ('Bob', 1, 90)
# ('Diana', 1, 85)
# ('Charlie', 2, 90)
# ('Alice', 2, 85)
```

### 4.2 Stable Sort

```python
# Python ใช้ Timsort ซึ่งเป็น stable sort
# สมาชิกที่มีค่า key เท่ากันจะรักษาลำดับเดิม

data = [("Alice", 30), ("Bob", 25), ("Charlie", 30), ("Diana", 25)]

# เรียงตามอายุ - ลำดับของคนอายุเท่ากันจะรักษาไว้
sorted_data = sorted(data, key=lambda x: x[1])
print(sorted_data)
# [('Bob', 25), ('Diana', 25), ('Alice', 30), ('Charlie', 30)]
# Bob และ Diana (อายุ 25) ยังอยู่ตามลำดับเดิม
```

---

## 5. Nested Lists (Matrix)

```python
# Nested list - list ของ list
matrix = [
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9]
]

# เข้าถึงสมาชิก
print(matrix[0])       # [1, 2, 3]   ← แถวแรก
print(matrix[1][2])    # 6           ← แถว 1, คอลัมน์ 2
print(matrix[2][0])    # 7

# Iterate nested list
for row in matrix:
    for val in row:
        print(val, end="\t")
    print()
# 1    2    3
# 4    5    6
# 7    8    9

# สร้าง matrix โดย loop
rows, cols = 3, 4
grid = [[0] * cols for _ in range(rows)]
print(grid)  # [[0, 0, 0, 0], [0, 0, 0, 0], [0, 0, 0, 0]]

# ⚠️ อย่าใช้ * สำหรับ nested list
wrong = [[0] * 4] * 3
wrong[0][0] = 99
print(wrong)  # [[99, 0, 0, 0], [99, 0, 0, 0], [99, 0, 0, 0]]  ← ผิด!

correct = [[0] * 4 for _ in range(3)]
correct[0][0] = 99
print(correct)  # [[99, 0, 0, 0], [0, 0, 0, 0], [0, 0, 0, 0]]  ← ถูก

# Transpose matrix
def transpose(matrix):
    """สลับแถวและคอลัมน์"""
    rows = len(matrix)
    cols = len(matrix[0])
    return [[matrix[r][c] for r in range(rows)] for c in range(cols)]

m = [[1, 2, 3], [4, 5, 6]]
print(transpose(m))  # [[1, 4], [2, 5], [3, 6]]

# หรือใช้ zip
transposed = [list(row) for row in zip(*m)]
print(transposed)  # [[1, 4], [2, 5], [3, 6]]

# Flatten nested list
nested = [[1, 2, 3], [4, 5], [6, 7, 8, 9]]
flat = [item for sublist in nested for item in sublist]
print(flat)  # [1, 2, 3, 4, 5, 6, 7, 8, 9]

# หรือใช้ itertools
import itertools
flat2 = list(itertools.chain.from_iterable(nested))
print(flat2)  # [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

---

## 6. List Comprehension ขั้นสูง

```python
# พื้นฐาน
squares = [x**2 for x in range(1, 11)]
print(squares)  # [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]

# กับเงื่อนไข (if)
evens = [x for x in range(20) if x % 2 == 0]
print(evens)  # [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]

# if-else ใน expression
labels = ["even" if x % 2 == 0 else "odd" for x in range(8)]
print(labels)  # ['even', 'odd', 'even', 'odd', 'even', 'odd', 'even', 'odd']

# หลาย for loop
pairs = [(x, y) for x in range(3) for y in range(3) if x != y]
print(pairs)  # [(0, 1), (0, 2), (1, 0), (1, 2), (2, 0), (2, 1)]

# Nested comprehension (matrix)
matrix = [[i * j for j in range(1, 6)] for i in range(1, 6)]
for row in matrix:
    print(row)
# [1, 2, 3, 4, 5]
# [2, 4, 6, 8, 10]
# ...

# กับฟังก์ชัน
def square(x): return x ** 2
results = [square(x) for x in range(1, 6)]
print(results)  # [1, 4, 9, 16, 25]

# กรอง None และ False
data = [1, None, 2, False, 3, "", 4, 0, 5]
clean = [x for x in data if x]
print(clean)  # [1, 2, 3, 4, 5]

# ตัวอย่างจริง: แปลง CSV string
csv_line = "Alice,30,Bangkok,Developer"
fields = [field.strip() for field in csv_line.split(",")]
print(fields)  # ['Alice', '30', 'Bangkok', 'Developer']
```

---

## 7. List Operations เพิ่มเติม

```python
# Concatenation (+)
a = [1, 2, 3]
b = [4, 5, 6]
c = a + b
print(c)  # [1, 2, 3, 4, 5, 6]

# Repetition (*)
zeros = [0] * 5
print(zeros)  # [0, 0, 0, 0, 0]

hello = ["ha"] * 3
print(hello)  # ['ha', 'ha', 'ha']

# Unpacking
first, *rest = [1, 2, 3, 4, 5]
print(first)  # 1
print(rest)   # [2, 3, 4, 5]

*init, last = [1, 2, 3, 4, 5]
print(init)   # [1, 2, 3, 4]
print(last)   # 5

first, *middle, last = [1, 2, 3, 4, 5]
print(first)   # 1
print(middle)  # [2, 3, 4]
print(last)    # 5

# zip() - รวม 2 list
names = ["Alice", "Bob", "Charlie"]
scores = [85, 92, 78]

for name, score in zip(names, scores):
    print(f"{name}: {score}")
# Alice: 85
# Bob: 92
# Charlie: 78

pairs = list(zip(names, scores))
print(pairs)  # [('Alice', 85), ('Bob', 92), ('Charlie', 78)]

# enumerate() - index + value
fruits = ["apple", "banana", "cherry"]
for i, fruit in enumerate(fruits):
    print(f"{i}: {fruit}")
# 0: apple
# 1: banana
# 2: cherry

for i, fruit in enumerate(fruits, start=1):
    print(f"{i}. {fruit}")
# 1. apple
# 2. banana
# 3. cherry

# any() และ all()
numbers = [2, 4, 6, 8, 10]
print(all(x % 2 == 0 for x in numbers))  # True (ทุกตัวเป็นเลขคู่)
print(any(x > 9 for x in numbers))        # True (มีบางตัว > 9)

scores = [85, 90, 72, 95, 88]
print(all(s >= 70 for s in scores))   # True (ทุกคนผ่าน 70)
print(any(s >= 90 for s in scores))   # True (มีใครได้ >= 90)

# max() min() sum() with key
students = [
    {"name": "Alice", "grade": 85},
    {"name": "Bob", "grade": 92},
    {"name": "Charlie", "grade": 78},
]

best = max(students, key=lambda s: s["grade"])
worst = min(students, key=lambda s: s["grade"])
total = sum(s["grade"] for s in students)

print(f"คะแนนสูงสุด: {best['name']} ({best['grade']})")
print(f"คะแนนต่ำสุด: {worst['name']} ({worst['grade']})")
print(f"คะแนนรวม: {total}, เฉลี่ย: {total/len(students):.1f}")
```

---

## 8. ตัวอย่างโปรแกรมจริง: ระบบจัดการนักเรียน

```python
# student_manager.py
"""ระบบจัดการข้อมูลนักเรียนอย่างง่าย"""

def create_student(name, grade, subjects):
    """สร้างข้อมูลนักเรียน"""
    return {
        "name": name,
        "grade": grade,
        "subjects": subjects,
        "average": sum(subjects) / len(subjects) if subjects else 0
    }

def add_student(students, name, grade, subjects):
    """เพิ่มนักเรียนใหม่"""
    student = create_student(name, grade, subjects)
    students.append(student)
    return students

def remove_student(students, name):
    """ลบนักเรียนตามชื่อ"""
    for i, s in enumerate(students):
        if s["name"] == name:
            students.pop(i)
            return True
    return False

def get_top_students(students, n=3):
    """คืน n นักเรียนที่มีคะแนนเฉลี่ยสูงสุด"""
    return sorted(students, key=lambda s: s["average"], reverse=True)[:n]

def get_failing_students(students, passing_grade=60):
    """คืนนักเรียนที่ตก"""
    return [s for s in students if s["average"] < passing_grade]

def get_class_stats(students):
    """คำนวณสถิติชั้นเรียน"""
    if not students:
        return None
    
    averages = [s["average"] for s in students]
    return {
        "count": len(students),
        "class_average": sum(averages) / len(averages),
        "highest": max(averages),
        "lowest": min(averages),
        "passing": sum(1 for a in averages if a >= 60),
        "failing": sum(1 for a in averages if a < 60),
    }

# ทดสอบระบบ
students = []

# เพิ่มนักเรียน
add_student(students, "อลิสา", 10, [85, 90, 88, 92, 87])
add_student(students, "บอบ", 10, [72, 68, 75, 70, 65])
add_student(students, "ชาร์ลี", 10, [95, 98, 92, 97, 93])
add_student(students, "ไดอาน่า", 10, [55, 58, 50, 60, 52])
add_student(students, "อีแวน", 10, [88, 85, 90, 87, 91])

print("=== รายชื่อนักเรียนทั้งหมด ===")
for s in students:
    print(f"  {s['name']}: เฉลี่ย {s['average']:.1f}")

print("\n=== Top 3 นักเรียน ===")
for i, s in enumerate(get_top_students(students), 1):
    print(f"  {i}. {s['name']}: {s['average']:.1f}")

print("\n=== นักเรียนที่ตก ===")
failing = get_failing_students(students)
if failing:
    for s in failing:
        print(f"  {s['name']}: เฉลี่ย {s['average']:.1f}")
else:
    print("  ไม่มีนักเรียนตก")

stats = get_class_stats(students)
print(f"\n=== สถิติชั้นเรียน ===")
print(f"  จำนวนนักเรียน: {stats['count']}")
print(f"  คะแนนเฉลี่ยชั้น: {stats['class_average']:.1f}")
print(f"  สูงสุด: {stats['highest']:.1f}")
print(f"  ต่ำสุด: {stats['lowest']:.1f}")
print(f"  ผ่าน: {stats['passing']} คน, ตก: {stats['failing']} คน")
```

---

## 9. Performance Tips

```python
import time

# ทดสอบ append vs extend
def test_append():
    lst = []
    for i in range(100000):
        lst.append(i)
    return lst

def test_extend():
    lst = []
    lst.extend(range(100000))
    return lst

def test_list():
    return list(range(100000))

# extend และ list() เร็วกว่า loop append
start = time.time()
test_append()
print(f"append loop: {time.time()-start:.4f}s")

start = time.time()
test_extend()
print(f"extend:      {time.time()-start:.4f}s")

start = time.time()
test_list()
print(f"list():      {time.time()-start:.4f}s")

# ใช้ in สำหรับ list เทียบกับ set
big_list = list(range(100000))
big_set = set(range(100000))

start = time.time()
for _ in range(1000):
    _ = 99999 in big_list  # O(n)
print(f"list in: {time.time()-start:.4f}s")

start = time.time()
for _ in range(1000):
    _ = 99999 in big_set   # O(1)
print(f"set in:  {time.time()-start:.4f}s")
```

---

## 10. Exercises

### Exercise 1: Shopping Cart

```python
"""
สร้างระบบ Shopping Cart อย่างง่าย ที่:
1. เพิ่มสินค้าได้ (ชื่อ, ราคา, จำนวน)
2. ลบสินค้าออกจากตะกร้าได้
3. คำนวณยอดรวมได้
4. แสดงรายการสินค้าเรียงตามราคา
"""

def create_cart():
    return []

def add_item(cart, name, price, quantity=1):
    # ตรวจสอบว่ามีสินค้านี้อยู่แล้วไหม
    for item in cart:
        if item["name"] == name:
            item["quantity"] += quantity
            return
    cart.append({"name": name, "price": price, "quantity": quantity})

def remove_item(cart, name):
    cart[:] = [item for item in cart if item["name"] != name]

def calculate_total(cart):
    return sum(item["price"] * item["quantity"] for item in cart)

def display_cart(cart):
    if not cart:
        print("ตะกร้าว่าง")
        return
    
    sorted_cart = sorted(cart, key=lambda x: x["price"], reverse=True)
    print("=== ตะกร้าสินค้า ===")
    for item in sorted_cart:
        subtotal = item["price"] * item["quantity"]
        print(f"  {item['name']:20} {item['quantity']:3} x {item['price']:8.2f} = {subtotal:10.2f} บาท")
    print(f"{'รวมทั้งหมด':>40} {calculate_total(cart):10.2f} บาท")

# ทดสอบ
cart = create_cart()
add_item(cart, "ข้าวสาร", 25.50, 2)
add_item(cart, "น้ำมันพืช", 75.00, 1)
add_item(cart, "น้ำตาล", 30.00, 3)
add_item(cart, "เกลือ", 15.00, 2)
add_item(cart, "ข้าวสาร", 25.50, 1)  # เพิ่ม quantity

display_cart(cart)
print()
remove_item(cart, "เกลือ")
display_cart(cart)
```

### Exercise 2: Matrix Operations

```python
"""
สร้างฟังก์ชันสำหรับ matrix operations:
1. matrix_add(a, b) - บวก 2 matrix
2. matrix_multiply(a, b) - คูณ 2 matrix
3. matrix_transpose(m) - transpose
4. print_matrix(m) - แสดงผลสวยงาม
"""

def matrix_add(a, b):
    """บวก matrix 2 ตัวที่มีขนาดเท่ากัน"""
    if len(a) != len(b) or len(a[0]) != len(b[0]):
        raise ValueError("Matrix ขนาดไม่เท่ากัน")
    return [[a[i][j] + b[i][j] for j in range(len(a[0]))] for i in range(len(a))]

def matrix_multiply(a, b):
    """คูณ matrix a (m×n) กับ b (n×p) ได้ matrix (m×p)"""
    m, n, p = len(a), len(a[0]), len(b[0])
    if n != len(b):
        raise ValueError("ขนาด matrix ไม่สามารถคูณกันได้")
    return [[sum(a[i][k] * b[k][j] for k in range(n)) for j in range(p)] for i in range(m)]

def matrix_transpose(m):
    """สลับแถวและคอลัมน์"""
    return [[m[i][j] for i in range(len(m))] for j in range(len(m[0]))]

def print_matrix(m, title="Matrix"):
    print(f"\n{title}:")
    for row in m:
        print("  [" + "  ".join(f"{x:4}" for x in row) + "]")

# ทดสอบ
A = [[1, 2, 3], [4, 5, 6]]
B = [[7, 8, 9], [10, 11, 12]]
C = [[1, 2], [3, 4], [5, 6]]

print_matrix(A, "A")
print_matrix(B, "B")
print_matrix(matrix_add(A, B), "A + B")
print_matrix(matrix_multiply(A, C), "A × C")
print_matrix(matrix_transpose(A), "A^T")
```

### Exercise 3: การวิเคราะห์ข้อมูล

```python
"""
วิเคราะห์ข้อมูลอุณหภูมิรายวัน:
- หาอุณหภูมิสูงสุด/ต่ำสุด และวันที่เกิด
- คำนวณค่าเฉลี่ยรายสัปดาห์
- หาวันที่อุณหภูมิสูงกว่าค่าเฉลี่ย
- Smooth data ด้วย moving average
"""

temperatures = {
    "Mon": 32.5, "Tue": 33.1, "Wed": 31.8, "Thu": 34.2, "Fri": 35.0,
    "Sat": 33.7, "Sun": 32.9, "Mon2": 31.5, "Tue2": 30.8, "Wed2": 32.3,
    "Thu2": 33.9, "Fri2": 34.6, "Sat2": 35.1, "Sun2": 34.2,
}

days = list(temperatures.keys())
temps = list(temperatures.values())

# สถิติ
max_temp = max(temps)
min_temp = min(temps)
avg_temp = sum(temps) / len(temps)
max_day = days[temps.index(max_temp)]
min_day = days[temps.index(min_temp)]

print(f"อุณหภูมิสูงสุด: {max_temp}°C (วัน{max_day})")
print(f"อุณหภูมิต่ำสุด: {min_temp}°C (วัน{min_day})")
print(f"ค่าเฉลี่ย: {avg_temp:.1f}°C")

# วันที่สูงกว่าค่าเฉลี่ย
hot_days = [(day, temp) for day, temp in temperatures.items() if temp > avg_temp]
print(f"\nวันที่ร้อนกว่าเฉลี่ย ({avg_temp:.1f}°C):")
for day, temp in hot_days:
    print(f"  {day}: {temp}°C")

# Moving average (window=3)
def moving_average(data, window=3):
    return [
        sum(data[i:i+window]) / window
        for i in range(len(data) - window + 1)
    ]

ma = moving_average(temps, 3)
print(f"\nMoving average (3 วัน):")
for i, avg in enumerate(ma):
    print(f"  วัน {i+1}-{i+3}: {avg:.2f}°C")
```

---

## 11. สรุป Part 007

### สิ่งที่เรียนรู้:

✅ **append()** - เพิ่มสมาชิกท้าย list  
✅ **extend()** - เพิ่มหลายสมาชิกจาก iterable  
✅ **insert(i, x)** - แทรกที่ตำแหน่งที่กำหนด  
✅ **remove(x)** - ลบสมาชิกแรกที่มีค่า x  
✅ **pop(i)** - ลบและคืนค่าสมาชิกที่ index i  
✅ **sort(key, reverse)** - เรียงลำดับ in-place  
✅ **reverse()** - กลับลำดับ in-place  
✅ **copy()** - shallow copy  
✅ **clear()** - ลบทั้งหมด  
✅ **count(x)** - นับจำนวน x  
✅ **index(x)** - หาตำแหน่ง x  
✅ **Slicing** - [start:stop:step]  
✅ **sorted()** - คืน list ใหม่ที่เรียงแล้ว  
✅ **Nested Lists** - matrix operations  
✅ **List Comprehension** - การสร้าง list อย่างกระชับ  

### Quick Reference:

```python
lst = [3, 1, 4, 1, 5, 9, 2, 6]

lst.append(7)          # เพิ่มท้าย
lst.extend([8, 9])     # เพิ่มหลายตัว
lst.insert(0, 0)       # แทรกที่ index 0
lst.remove(1)          # ลบค่า 1 (ตัวแรก)
last = lst.pop()       # ลบและคืนค่าสุดท้าย
lst.sort()             # เรียง ascending
lst.sort(reverse=True) # เรียง descending
lst.reverse()          # กลับลำดับ
copy = lst.copy()      # คัดลอก
lst.clear()            # ล้างทั้งหมด
n = lst.count(5)       # นับ 5
i = lst.index(4)       # หา index ของ 4

# Slicing
lst[1:4]    # index 1, 2, 3
lst[::2]    # ทุก 2 ตัว
lst[::-1]   # กลับหลัง
```

---

## ➡️ ถัดไป: Part 008 - Tuples

ใน Part ถัดไป เราจะเรียนรู้:
- Tuple คืออะไร และทำไมถึงใช้
- Immutability และ use cases
- Packing/Unpacking
- Named Tuples
- Tuple vs List เมื่อไหรควรใช้อะไร

---

*Part 007/100+ | Python Course - Beginner to World-Class*
