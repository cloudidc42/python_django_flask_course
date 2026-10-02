# Part 003: ตัวดำเนินการและนิพจน์ (Operators & Expressions)
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- รู้จักตัวดำเนินการทุกประเภทใน Python
- เข้าใจลำดับความสำคัญของตัวดำเนินการ
- ใช้ Comparison และ Logical operators ได้
- เข้าใจ Bitwise operators
- ใช้ Assignment operators แบบย่อได้

---

## 1. Arithmetic Operators (ตัวดำเนินการคณิตศาสตร์)

```python
a = 17
b = 5

# พื้นฐาน
print(a + b)     # 22    - บวก (Addition)
print(a - b)     # 12    - ลบ (Subtraction)
print(a * b)     # 85    - คูณ (Multiplication)
print(a / b)     # 3.4   - หาร (Division) → ได้ float เสมอ
print(a // b)    # 3     - หารเต็ม (Floor Division)
print(a % b)     # 2     - เศษจากการหาร (Modulo)
print(a ** b)    # 1419857 - ยกกำลัง (Exponentiation)

# หารเต็มกับจำนวนลบ
print(7 // 2)    # 3     - ปัดลงเสมอ
print(-7 // 2)   # -4    - ปัดลง (ไม่ใช่ -3!)
print(7 // -2)   # -4    - ปัดลง
print(-7 // -2)  # 3     - ปัดลง

# Modulo กับจำนวนลบ
print(7 % 3)     # 1
print(-7 % 3)    # 2     (ผลลัพธ์เป็น sign ของตัวหาร)
print(7 % -3)    # -2
print(-7 % -3)   # -1

# การใช้งานจริง
# ตรวจสอบเลขคู่/คี่
for n in range(1, 11):
    if n % 2 == 0:
        print(f"{n} เป็นเลขคู่")
    else:
        print(f"{n} เป็นเลขคี่")

# แปลงวินาทีเป็นชั่วโมง/นาที/วินาที
total_seconds = 3723
hours = total_seconds // 3600
minutes = (total_seconds % 3600) // 60
seconds = total_seconds % 60
print(f"{total_seconds} วินาที = {hours}:{minutes:02d}:{seconds:02d}")  # 1:02:03

# ตรวจสอบปีอธิกสุรทิน (Leap Year)
def is_leap_year(year):
    return (year % 4 == 0 and year % 100 != 0) or (year % 400 == 0)

for year in [2000, 1900, 2024, 2023]:
    print(f"{year}: {'ปีอธิกสุรทิน' if is_leap_year(year) else 'ปีปกติ'}")
```

### 1.1 ตัวดำเนินการทางคณิตศาสตร์ขั้นสูง

```python
import math

# ยกกำลัง
print(2 ** 8)            # 256
print(2 ** 0.5)          # 1.4142... (รากที่สอง)
print(27 ** (1/3))       # 3.0 (รากที่สาม)

# math functions
print(math.sqrt(16))     # 4.0 (รากที่สอง)
print(math.pow(2, 8))    # 256.0 (float)
print(math.factorial(5)) # 120
print(math.gcd(48, 18))  # 6 (Greatest Common Divisor)
print(math.lcm(4, 6))    # 12 (Least Common Multiple) - Python 3.9+
print(math.log(100, 10)) # 2.0 (log base 10)
print(math.log2(8))      # 3.0 (log base 2)
print(math.log10(1000))  # 3.0

# Trigonometry
import math
angle_deg = 45
angle_rad = math.radians(angle_deg)
print(f"sin(45°) = {math.sin(angle_rad):.4f}")     # 0.7071
print(f"cos(45°) = {math.cos(angle_rad):.4f}")     # 0.7071
print(f"tan(45°) = {math.tan(angle_rad):.4f}")     # 1.0000

# Complex numbers
c1 = 3 + 4j
c2 = 1 - 2j
print(c1 + c2)    # (4+2j)
print(c1 * c2)    # (11-2j)
print(abs(c1))    # 5.0 (magnitude)
print(c1.real)    # 3.0
print(c1.imag)    # 4.0
```

---

## 2. Comparison Operators (ตัวดำเนินการเปรียบเทียบ)

```python
a = 10
b = 20

print(a == b)    # False  - เท่ากับ (Equal)
print(a != b)    # True   - ไม่เท่ากับ (Not Equal)
print(a < b)     # True   - น้อยกว่า (Less than)
print(a > b)     # False  - มากกว่า (Greater than)
print(a <= b)    # True   - น้อยกว่าหรือเท่ากับ
print(a >= b)    # False  - มากกว่าหรือเท่ากับ

# Chained comparisons (Python feature พิเศษ)
x = 15
print(10 < x < 20)          # True  (10 < 15 < 20)
print(0 <= x <= 100)         # True  (0 ≤ 15 ≤ 100)
print(10 < x > 5)            # True  (10 < 15 and 15 > 5)
print(1 < 2 < 3 < 4 < 5)    # True

# เปรียบเทียบ string (ตามลำดับ ASCII/Unicode)
print("apple" < "banana")    # True  (a < b)
print("apple" == "apple")    # True
print("Apple" < "apple")     # True  (A=65 < a=97)
print("abc" < "abd")         # True  (เปรียบเทียบทีละตัวอักษร)

# เปรียบเทียบ list
print([1, 2, 3] == [1, 2, 3])   # True
print([1, 2, 3] == [1, 2, 4])   # False
print([1, 2] < [1, 3])          # True  (เปรียบเทียบทีละ element)
print([1, 2, 3] < [1, 2, 3, 4]) # True  (list สั้นกว่า < list ยาวกว่า ถ้าเท่ากัน)

# is vs == 
a = [1, 2, 3]
b = [1, 2, 3]
c = a

print(a == b)    # True  (ค่าเหมือนกัน)
print(a is b)    # False (คนละ object)
print(a is c)    # True  (เป็น object เดียวกัน)
print(id(a), id(b), id(c))  # a และ c มี id เดียวกัน

# ใช้ is สำหรับ None เสมอ
x = None
print(x is None)     # ✅ แนะนำ
print(x == None)     # ⚠️ ใช้ได้แต่ไม่แนะนำ

# in operator
fruits = ["apple", "banana", "cherry"]
print("apple" in fruits)        # True
print("grape" in fruits)        # False
print("grape" not in fruits)    # True

text = "Hello, World!"
print("World" in text)          # True
print("Python" in text)         # False

# is vs is not
x = None
print(x is None)        # True
print(x is not None)    # False

numbers = [1, 2, 3]
print(numbers is not None)  # True
```

---

## 3. Logical Operators (ตัวดำเนินการตรรกะ)

```python
# and, or, not
a = True
b = False

print(a and b)      # False
print(a or b)       # True
print(not a)        # False
print(not b)        # True

# Truth Table: and
print("\nTruth Table: AND")
for x in [True, False]:
    for y in [True, False]:
        print(f"  {x} and {y} = {x and y}")

# Truth Table: or
print("\nTruth Table: OR")
for x in [True, False]:
    for y in [True, False]:
        print(f"  {x} or {y} = {x or y}")

# การใช้งานจริง
age = 25
income = 50000
has_id = True

# ตรวจสอบสิทธิ์กู้เงิน
can_apply_loan = age >= 20 and income >= 30000 and has_id
print(f"สมัครกู้ได้: {can_apply_loan}")

# ตรวจสอบเข้าคลับ
is_member = False
is_vip = True
can_enter = is_member or is_vip
print(f"เข้าคลับได้: {can_enter}")

# ปฏิเสธ
is_banned = False
can_play = not is_banned
print(f"เล่นได้: {can_play}")

# Short-circuit evaluation
print("\n--- Short-circuit ---")

def check_something():
    print("ฟังก์ชันถูกเรียก!")
    return True

# and: ถ้าตัวแรก False ไม่เรียกตัวที่สอง
print(False and check_something())   # False (ไม่เรียก check_something)
print(True and check_something())    # เรียก check_something แล้วได้ True

# or: ถ้าตัวแรก True ไม่เรียกตัวที่สอง
print(True or check_something())     # True (ไม่เรียก check_something)
print(False or check_something())    # เรียก check_something แล้วได้ True

# ใช้ประโยชน์จาก short-circuit
user = None
# วิธีปลอดภัยตรวจสอบ None ก่อนใช้งาน
user_name = user and user.get('name')  # ถ้า user เป็น None จะได้ None ทันที
print(user_name)  # None

user = {"name": "Alice", "age": 25}
user_name = user and user.get('name')  # ถ้า user มีค่า จะ get name
print(user_name)  # Alice

# Default value ด้วย or
config = {}
host = config.get('host') or 'localhost'
port = config.get('port') or 8080
print(f"{host}:{port}")  # localhost:8080

config = {'host': '192.168.1.1', 'port': 3000}
host = config.get('host') or 'localhost'
port = config.get('port') or 8080
print(f"{host}:{port}")  # 192.168.1.1:3000
```

### 3.1 Logical Operators กับ Non-Boolean Values

```python
# and ส่งคืน: ถ้า falsy ส่งตัวแรก, ถ้า truthy ส่งตัวสุดท้าย
print(1 and 2)       # 2   (ทั้งคู่ truthy → ส่งตัวสุดท้าย)
print(0 and 2)       # 0   (ตัวแรก falsy → ส่งตัวแรก)
print("" and "abc")  # ""  (ตัวแรก falsy → ส่งตัวแรก)
print("a" and "b")   # "b" (ทั้งคู่ truthy → ส่งตัวสุดท้าย)

# or ส่งคืน: ถ้า truthy ส่งตัวแรก, ถ้า falsy ส่งตัวสุดท้าย
print(1 or 2)        # 1   (ตัวแรก truthy → ส่งตัวแรก)
print(0 or 2)        # 2   (ตัวแรก falsy → ส่งตัวที่สอง)
print("" or "abc")   # "abc" (ตัวแรก falsy → ส่งตัวที่สอง)
print(None or [])    # []  (ทั้งคู่ falsy → ส่งตัวสุดท้าย)

# ประยุกต์ใช้
def get_username(user):
    return user.get('name') or user.get('email') or 'Unknown'

user1 = {'name': 'Alice', 'email': 'alice@example.com'}
user2 = {'email': 'bob@example.com'}
user3 = {}

print(get_username(user1))  # Alice
print(get_username(user2))  # bob@example.com
print(get_username(user3))  # Unknown
```

---

## 4. Assignment Operators (ตัวดำเนินการกำหนดค่า)

```python
# กำหนดค่าพื้นฐาน
x = 10
print(x)     # 10

# Augmented Assignment Operators
x = 10
x += 5       # x = x + 5
print(x)     # 15

x = 10
x -= 3       # x = x - 3
print(x)     # 7

x = 10
x *= 2       # x = x * 2
print(x)     # 20

x = 10
x /= 4       # x = x / 4
print(x)     # 2.5

x = 10
x //= 3      # x = x // 3
print(x)     # 3

x = 10
x %= 3       # x = x % 3
print(x)     # 1

x = 2
x **= 4      # x = x ** 4
print(x)     # 16

# กับ string
s = "Hello"
s += " World"
print(s)     # Hello World

s = "Ha"
s *= 3
print(s)     # HaHaHa

# กับ list
lst = [1, 2]
lst += [3, 4]
print(lst)   # [1, 2, 3, 4]

lst = [1, 2]
lst *= 3
print(lst)   # [1, 2, 1, 2, 1, 2]

# Walrus Operator := (Assignment Expression, Python 3.8+)
import re

# แบบเก่า
data = "12345"
match = re.match(r'\d+', data)
if match:
    print(f"Found: {match.group()}")

# ใช้ Walrus Operator
if match := re.match(r'\d+', data):
    print(f"Found: {match.group()}")

# ใน while loop
while chunk := "some data":    # อ่านข้อมูล
    break                       # จบ loop

# ใน list comprehension
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
even_squares = [sq for n in numbers if (sq := n**2) > 25]
print(even_squares)  # [36, 49, 64, 81, 100]
```

---

## 5. Bitwise Operators (ตัวดำเนินการบิต)

```python
# Bitwise operators ทำงานระดับบิต
a = 0b1100   # 12 ในฐานสิบ
b = 0b1010   # 10 ในฐานสิบ

# AND (&) - บิต 1 ก็ต่อเมื่อทั้งคู่เป็น 1
print(f"a & b = {a & b:04b} ({a & b})")    # 1000 (8)

# OR (|) - บิต 1 ถ้าอย่างน้อยหนึ่งตัวเป็น 1
print(f"a | b = {a | b:04b} ({a | b})")    # 1110 (14)

# XOR (^) - บิต 1 ถ้าต่างกัน
print(f"a ^ b = {a ^ b:04b} ({a ^ b})")    # 0110 (6)

# NOT (~) - กลับบิตทั้งหมด
print(f"~a = {~a} ({bin(~a)})")              # -13

# Left Shift (<<) - เลื่อนบิตไปซ้าย (คูณ 2)
print(f"a << 1 = {a << 1:08b} ({a << 1})")  # 11000 (24)
print(f"a << 2 = {a << 2} = {a} × 4")       # 48

# Right Shift (>>) - เลื่อนบิตไปขวา (หาร 2)
print(f"a >> 1 = {a >> 1:04b} ({a >> 1})")  # 0110 (6)
print(f"a >> 2 = {a >> 2} = {a} ÷ 4")       # 3

# การใช้งานจริง
# 1. ตรวจสอบเลขคู่/คี่ (เร็วกว่า %)
def is_even_bitwise(n):
    return (n & 1) == 0

for i in range(6):
    print(f"{i}: {'คู่' if is_even_bitwise(i) else 'คี่'}")

# 2. ยกกำลัง 2
for i in range(8):
    print(f"2^{i} = {1 << i}")

# 3. Permission flags (เหมือน Linux permissions)
READ    = 0b001  # 1
WRITE   = 0b010  # 2
EXECUTE = 0b100  # 4

# กำหนด permissions
user_perms = READ | WRITE       # 011 = 3
print(f"User permissions: {bin(user_perms)}")

# ตรวจสอบ permission
print(f"Can read: {bool(user_perms & READ)}")       # True
print(f"Can write: {bool(user_perms & WRITE)}")     # True
print(f"Can execute: {bool(user_perms & EXECUTE)}") # False

# เพิ่ม permission
user_perms |= EXECUTE
print(f"After adding execute: {bin(user_perms)}")   # 0b111

# ลบ permission
user_perms &= ~EXECUTE
print(f"After removing execute: {bin(user_perms)}") # 0b011

# Toggle permission
user_perms ^= WRITE
print(f"After toggling write: {bin(user_perms)}")   # 0b001

# 4. RGB Color manipulation
def get_rgb(color_hex):
    r = (color_hex >> 16) & 0xFF
    g = (color_hex >> 8) & 0xFF
    b = color_hex & 0xFF
    return r, g, b

color = 0xFF5733  # RGB(255, 87, 51)
r, g, b = get_rgb(color)
print(f"R={r}, G={g}, B={b}")

def rgb_to_hex(r, g, b):
    return (r << 16) | (g << 8) | b

hex_color = rgb_to_hex(255, 87, 51)
print(f"Hex: #{hex_color:06X}")
```

---

## 6. Identity Operators

```python
# is - ตรวจสอบว่าเป็น object เดียวกัน (same memory address)
# is not - ตรวจสอบว่าไม่ใช่ object เดียวกัน

a = [1, 2, 3]
b = [1, 2, 3]
c = a

print(a is c)       # True  (a และ c ชี้ไป object เดียวกัน)
print(a is b)       # False (a และ b เป็น list คนละตัว แม้ค่าเหมือนกัน)
print(a is not b)   # True

# None ใช้ is เสมอ
x = None
print(x is None)     # ✅
print(x is not None) # False

# Integer caching
# Python cache integers -5 ถึง 256
a = 100
b = 100
print(a is b)    # True  (cached)

a = 1000
b = 1000
print(a is b)    # False (อาจ False สำหรับ large integers)
print(a == b)    # True  (ค่าเท่ากัน)

# String interning
s1 = "hello"
s2 = "hello"
print(s1 is s2)  # True  (Python intern short strings)

s1 = "hello world with many words"
s2 = "hello world with many words"
print(s1 is s2)  # อาจ True หรือ False
print(s1 == s2)  # True เสมอ
```

---

## 7. Membership Operators

```python
# in, not in
fruits = ["apple", "banana", "cherry", "date"]

print("apple" in fruits)        # True
print("grape" in fruits)        # False
print("grape" not in fruits)    # True

# ใน string
text = "Hello, World!"
print("Hello" in text)          # True
print("Python" in text)         # False
print("Python" not in text)     # True

# ใน dictionary (ตรวจสอบ keys)
person = {"name": "Alice", "age": 25, "city": "Bangkok"}
print("name" in person)         # True  (เช็ค key)
print("Alice" in person)        # False (ไม่เช็ค value โดยตรง)
print("Alice" in person.values())  # True (เช็ค values)

# ใน set
vowels = {'a', 'e', 'i', 'o', 'u'}
print('a' in vowels)            # True
print('b' in vowels)            # False

# ใน range
print(5 in range(1, 10))        # True
print(10 in range(1, 10))       # False  (10 ไม่อยู่ใน 1-9)
print(10 in range(1, 11))       # True

# ตัวอย่างใช้งานจริง
def classify_char(char):
    if char in 'aeiouAEIOU':
        return "สระ"
    elif char in 'bcdfghjklmnpqrstvwxyzBCDFGHJKLMNPQRSTVWXYZ':
        return "พยัญชนะ"
    elif char in '0123456789':
        return "ตัวเลข"
    else:
        return "อักขระพิเศษ"

for c in "Hello!123":
    print(f"'{c}' = {classify_char(c)}")
```

---

## 8. Operator Precedence (ลำดับความสำคัญ)

```python
# Python ประเมินตามลำดับความสำคัญ (สูงไปต่ำ):
"""
1.  ()                    - Parentheses (วงเล็บ)
2.  **                    - Exponentiation (ยกกำลัง)
3.  +x, -x, ~x           - Unary (เครื่องหมาย)
4.  *, /, //, %          - Multiplication/Division
5.  +, -                  - Addition/Subtraction
6.  <<, >>                - Bitwise Shift
7.  &                     - Bitwise AND
8.  ^                     - Bitwise XOR
9.  |                     - Bitwise OR
10. ==, !=, >, <, >=, <=, is, is not, in, not in  - Comparison
11. not                   - Logical NOT
12. and                   - Logical AND
13. or                    - Logical OR
14. :=                    - Walrus operator
"""

# ตัวอย่าง
print(2 + 3 * 4)         # 14  (คูณก่อน บวกทีหลัง)
print((2 + 3) * 4)       # 20  (วงเล็บก่อน)
print(2 ** 3 ** 2)       # 512 (** จากขวา: 2**(3**2) = 2**9)
print((2 ** 3) ** 2)     # 64  ((2**3)**2 = 8**2)

print(True or False and False)   # True  (and ก่อน or)
print((True or False) and False) # False

print(not True or False)   # False (not ก่อน or)
print(not (True or False)) # False

# การใช้วงเล็บทำให้ชัดเจน (แนะนำ)
x = 5
result = (x > 3) and (x < 10)   # ✅ ชัดเจน
result = x > 3 and x < 10        # ✅ ก็ทำงานถูก แต่ชัดกว่าใส่วงเล็บ

# ตัวอย่างซับซ้อน
a, b, c = 2, 3, 4
print(a + b * c)              # 14  (3*4 ก่อน แล้วบวก 2)
print(a ** b ** c)            # มหาศาล (b**c ก่อน: 3**4=81, แล้ว 2**81)
print(-a ** 2)                # -4  (ยกกำลังก่อน เครื่องหมาย - หลังสุด)
print((-a) ** 2)              # 4   (ใส่วงเล็บถ้าต้องการ)
```

---

## 9. Conditional Expression (Ternary Operator)

```python
# รูปแบบ: value_if_true if condition else value_if_false
age = 20
status = "ผู้ใหญ่" if age >= 18 else "เด็ก"
print(status)   # ผู้ใหญ่

# แบบ if/else ปกติ
if age >= 18:
    status = "ผู้ใหญ่"
else:
    status = "เด็ก"

# Nested ternary (ไม่แนะนำให้ใช้ซ้อนกันมากเกินไป)
score = 75
grade = "A" if score >= 90 else "B" if score >= 80 else "C" if score >= 70 else "F"
print(grade)    # C

# ใน print
x = 10
print(f"x is {'positive' if x > 0 else 'negative' if x < 0 else 'zero'}")

# ใน list comprehension
numbers = range(-5, 6)
results = ["pos" if n > 0 else "neg" if n < 0 else "zero" for n in numbers]
print(results)

# ใน function argument
data = [1, 2, 3, 4, 5]
print(max(data) if data else None)   # 5
print(max([]) if [] else None)       # None
```

---

## 10. Augmented Assignment ขั้นสูง

```python
# Augmented bitwise assignment
x = 0b1100   # 12
x &= 0b1010  # x = x & 0b1010
print(bin(x))  # 0b1000 (8)

x = 0b1100
x |= 0b1010
print(bin(x))  # 0b1110 (14)

x = 0b1100
x ^= 0b1010
print(bin(x))  # 0b110 (6)

x = 12
x >>= 1      # x = x >> 1
print(x)     # 6

x = 12
x <<= 2      # x = x << 2
print(x)     # 48

# กับ strings
greeting = "Hello"
greeting += ", World!"
greeting += " How are you?"
print(greeting)   # Hello, World! How are you?

# กับ lists
items = []
items += ["first"]
items += ["second", "third"]
print(items)  # ['first', 'second', 'third']

# Counter pattern
word = "hello world"
char_count = {}
for char in word:
    if char in char_count:
        char_count[char] += 1
    else:
        char_count[char] = 1

print(char_count)
# {'h': 1, 'e': 1, 'l': 3, 'o': 2, ' ': 1, 'w': 1, 'r': 1, 'd': 1}
```

---

## 11. ตัวอย่างโปรแกรม: เครื่องคิดเลขขั้นสูง

```python
# calculator.py
"""
เครื่องคิดเลขที่ใช้ operators ต่างๆ
"""

def calculate(a, b, operator):
    """คำนวณผลลัพธ์จาก operator ที่กำหนด"""
    operations = {
        '+':  lambda x, y: x + y,
        '-':  lambda x, y: x - y,
        '*':  lambda x, y: x * y,
        '/':  lambda x, y: x / y if y != 0 else None,
        '//': lambda x, y: x // y if y != 0 else None,
        '%':  lambda x, y: x % y if y != 0 else None,
        '**': lambda x, y: x ** y,
    }
    
    if operator not in operations:
        return f"Error: operator '{operator}' ไม่รู้จัก"
    
    result = operations[operator](a, b)
    if result is None:
        return "Error: หารด้วยศูนย์"
    return result

def format_number(n):
    """จัดรูปแบบตัวเลข"""
    if isinstance(n, float) and n.is_integer():
        return int(n)
    elif isinstance(n, float):
        return round(n, 6)
    return n

# ทดสอบ
test_cases = [
    (10, 3, '+'),
    (10, 3, '-'),
    (10, 3, '*'),
    (10, 3, '/'),
    (10, 3, '//'),
    (10, 3, '%'),
    (2, 10, '**'),
    (10, 0, '/'),
]

print("=" * 50)
print("         เครื่องคิดเลขอัตโนมัติ")
print("=" * 50)

for a, b, op in test_cases:
    result = calculate(a, b, op)
    if isinstance(result, str):  # Error message
        print(f"  {a:5} {op:3} {b:2}  =  {result}")
    else:
        print(f"  {a:5} {op:3} {b:2}  =  {format_number(result)}")

print("=" * 50)

# ตรวจสอบเลขพิเศษ
import math
print("\n--- เลขพิเศษ ---")
primes = [n for n in range(2, 50) if all(n % i != 0 for i in range(2, int(math.sqrt(n)) + 1))]
print(f"เลขเฉพาะ 1-50: {primes}")

# Fibonacci ด้วย operator
def fibonacci(n):
    if n <= 1:
        return n
    a, b = 0, 1
    for _ in range(2, n + 1):
        a, b = b, a + b     # swap + add ในบรรทัดเดียว
    return b

fib_sequence = [fibonacci(i) for i in range(15)]
print(f"Fibonacci (15 ตัว): {fib_sequence}")
```

---

## 12. Exercises (แบบฝึกหัด)

### Exercise 1: เครื่องแปลงหน่วย
```python
# สร้างเครื่องแปลงหน่วยที่ใช้ operators ต่างๆ

temperature_c = 37.0  # องศาเซลเซียส

# แปลงเป็น Fahrenheit: F = C × 9/5 + 32
temperature_f = temperature_c * 9 / 5 + 32

# แปลงเป็น Kelvin: K = C + 273.15
temperature_k = temperature_c + 273.15

print(f"อุณหภูมิ:")
print(f"  Celsius:    {temperature_c}°C")
print(f"  Fahrenheit: {temperature_f:.1f}°F")
print(f"  Kelvin:     {temperature_k:.2f}K")
```

### Exercise 2: Bitwise Operations
```python
# ทดลอง bitwise operators กับ permissions

# กำหนด flags
ADMIN    = 1 << 0    # 00000001 = 1
USER     = 1 << 1    # 00000010 = 2
STAFF    = 1 << 2    # 00000100 = 4
PREMIUM  = 1 << 3    # 00001000 = 8

# สร้าง user ที่มี permissions ต่างกัน
alice = ADMIN | USER | PREMIUM    # Alice เป็น admin, user, premium
bob = USER | STAFF                # Bob เป็น user, staff
charlie = USER                    # Charlie เป็นแค่ user

def check_permission(user_perms, permission, permission_name):
    has = bool(user_perms & permission)
    print(f"  {permission_name}: {'✅' if has else '❌'}")

def show_permissions(name, perms):
    print(f"\n{name} permissions:")
    check_permission(perms, ADMIN, "Admin")
    check_permission(perms, USER, "User")
    check_permission(perms, STAFF, "Staff")
    check_permission(perms, PREMIUM, "Premium")

show_permissions("Alice", alice)
show_permissions("Bob", bob)
show_permissions("Charlie", charlie)
```

### Exercise 3: Logical Operators
```python
# สร้างระบบตรวจสอบการสมัครสมาชิก

def can_register(age, email, password, agree_terms):
    """
    ตรวจสอบว่าสามารถสมัครสมาชิกได้หรือไม่
    - อายุ >= 13
    - email ต้องมี @ และ .
    - password >= 8 ตัวอักษร
    - ต้องยอมรับ terms
    """
    age_ok = age >= 13
    email_ok = '@' in email and '.' in email
    password_ok = len(password) >= 8
    terms_ok = agree_terms
    
    all_ok = age_ok and email_ok and password_ok and terms_ok
    
    if not all_ok:
        print("ไม่สามารถสมัครได้:")
        print(f"  อายุ: {'✅' if age_ok else '❌ (ต้องอายุ >= 13)'}")
        print(f"  Email: {'✅' if email_ok else '❌ (รูปแบบไม่ถูกต้อง)'}")
        print(f"  Password: {'✅' if password_ok else '❌ (ต้องมีอย่างน้อย 8 ตัว)'}")
        print(f"  Terms: {'✅' if terms_ok else '❌ (ต้องยอมรับ terms)'}")
    else:
        print("สมัครสมาชิกสำเร็จ! ✅")
    
    return all_ok

# ทดสอบ
print("=== Test Case 1 ===")
can_register(25, "alice@email.com", "password123", True)

print("\n=== Test Case 2 ===")
can_register(10, "bob@email.com", "pass", True)

print("\n=== Test Case 3 ===")
can_register(18, "charlie@", "securepass123", False)
```

---

## 13. สรุป Part 003

### สิ่งที่เรียนรู้:

✅ **Arithmetic Operators** - +, -, *, /, //, %, **  
✅ **Comparison Operators** - ==, !=, <, >, <=, >=  
✅ **Logical Operators** - and, or, not  
✅ **Assignment Operators** - =, +=, -=, *=, /=, //=, %=, **=  
✅ **Bitwise Operators** - &, |, ^, ~, <<, >>  
✅ **Identity Operators** - is, is not  
✅ **Membership Operators** - in, not in  
✅ **Operator Precedence** - ลำดับความสำคัญ  
✅ **Ternary/Conditional Expression** - x if condition else y  

### Quick Reference:

```python
# Arithmetic
17 // 5  = 3    # floor division
17 % 5   = 2    # modulo (remainder)
2 ** 8   = 256  # power

# Comparison
a == b   # equal
a is b   # same object
"x" in lst  # membership

# Logical (short-circuit)
x = None
name = x or "default"      # "default"
value = x and x.value      # None (safe)

# Bitwise
x & y    # AND
x | y    # OR
x ^ y    # XOR
x << n   # left shift (× 2^n)
x >> n   # right shift (÷ 2^n)

# Ternary
"pass" if score >= 60 else "fail"
```

---

## ➡️ ถัดไป: Part 004 - การควบคุมการไหลของโปรแกรม (if/elif/else)

ใน Part ถัดไป เราจะเรียนรู้:
- if, elif, else statements
- Nested conditions
- Match/Case (Python 3.10+)
- Practical examples

---

*Part 003/100+ | Python Course - Beginner to World-Class*
