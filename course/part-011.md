# Part 011: Strings ขั้นสูง
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ f-strings ขั้นสูงพร้อม format spec ได้
- ใช้ String Methods ได้ครบทุก method
- ใช้ Regular Expressions พื้นฐานได้
- เข้าใจ String Encoding และ Bytes ได้
- ใช้ Multiline Strings และ Raw Strings ได้

---

## 1. String พื้นฐานทบทวน

```python
# String literals
s1 = 'single quotes'
s2 = "double quotes"
s3 = """triple double
quotes (multiline)"""
s4 = '''triple single
quotes (multiline)'''

# Raw strings - \ ไม่ escape
path = r"C:\Users\Alice\Documents"
pattern = r"\d+\.\d+"   # regex pattern

print(path)     # C:\Users\Alice\Documents
print(r"\n")    # \n  (ไม่ใช่ newline)
print("\n")     # (newline)

# Bytes
b = b"Hello"
print(type(b))   # <class 'bytes'>
print(b[0])      # 72 (ASCII of 'H')

# Unicode
emoji = "สวัสดี 🐍"
print(emoji)
print(len(emoji))     # 10
print(len(emoji.encode("utf-8")))  # 25 (bytes)

# String ไม่เปลี่ยนแปลง (immutable)
s = "hello"
try:
    s[0] = "H"
except TypeError as e:
    print(f"Error: {e}")

# สร้าง string ใหม่แทน
s = "H" + s[1:]  # "Hello"
print(s)
```

---

## 2. String Formatting ขั้นสูง

### 2.1 f-strings (Python 3.6+)

```python
name = "Alice"
age = 30
pi = 3.14159265358979

# พื้นฐาน
print(f"ชื่อ: {name}, อายุ: {age}")

# Expressions
print(f"2 + 2 = {2 + 2}")
print(f"age * 2 = {age * 2}")
print(f"name ตัวใหญ่: {name.upper()}")

# Multi-line f-string
message = (
    f"สวัสดี {name}! "
    f"คุณอายุ {age} ปี "
    f"และ pi ≈ {pi:.4f}"
)
print(message)

# f-string ใน f-string (Python 3.12+)
width = 10
print(f"{'hello':>{width}}")  # ชิดขวา 10 chars

# Conditional ใน f-string
score = 85
grade = f"{'A' if score >= 90 else 'B' if score >= 80 else 'C'}"
print(f"คะแนน {score}: เกรด {grade}")

# Dictionary ใน f-string
person = {"name": "Bob", "age": 25}
print(f"Name: {person['name']}, Age: {person['age']}")

# ใช้ " ใน f-string
print(f'He said "Hello"')  # ใช้ ' ข้างนอก
print(f"She said {'Alice'!r}")  # !r เพิ่ม quotes

# Debug mode (Python 3.8+) - = แสดง ชื่อตัวแปร
x = 42
print(f"{x = }")           # x = 42
print(f"{x * 2 = }")       # x * 2 = 84
print(f"{name.upper() = }") # name.upper() = 'ALICE'
```

### 2.2 Format Spec Mini-Language

```python
# f"{value:format_spec}"
# Format: [[fill]align][sign][#][0][width][grouping_option][.precision][type]

# Width
print(f"{'hello':10}")   # 'hello     '  ← ชิดซ้าย (default)
print(f"{'hello':>10}")  # '     hello'  ← ชิดขวา
print(f"{'hello':^10}")  # '  hello   '  ← ตรงกลาง
print(f"{'hello':<10}")  # 'hello     '  ← ชิดซ้าย explicit

# Fill character
print(f"{'hello':*>10}")  # '*****hello'
print(f"{'hello':*<10}")  # 'hello*****'
print(f"{'hello':*^10}")  # '**hello***'
print(f"{42:0>5}")        # '00042'

# Numbers
n = 1234567.891234

print(f"{n:.2f}")         # 1234567.89   (2 decimal places)
print(f"{n:,.2f}")        # 1,234,567.89 (thousands separator)
print(f"{n:e}")           # 1.234568e+06 (scientific)
print(f"{n:.2e}")         # 1.23e+06
print(f"{n:g}")           # 1.23457e+06 (general)

# Integer formats
n = 255
print(f"{n:d}")    # 255    (decimal)
print(f"{n:b}")    # 11111111 (binary)
print(f"{n:o}")    # 377    (octal)
print(f"{n:x}")    # ff     (hex lowercase)
print(f"{n:X}")    # FF     (hex uppercase)
print(f"{n:#x}")   # 0xff   (hex with prefix)
print(f"{n:#b}")   # 0b11111111 (binary with prefix)
print(f"{n:08b}")  # 11111111 (8-digit binary)
print(f"{n:08x}")  # 000000ff (8-digit hex)

# Percentage
ratio = 0.8543
print(f"{ratio:.1%}")   # 85.4%
print(f"{ratio:.2%}")   # 85.43%

# Sign
print(f"{42:+}")    # +42
print(f"{-42:+}")   # -42
print(f"{42: }")    # ' 42' (space for positive)
print(f"{-42: }")   # '-42'

# Nested format spec
precision = 4
print(f"{pi:.{precision}f}")  # 3.1416

# ตัวอย่าง: ตาราง
headers = ["ชื่อ", "อายุ", "คะแนน", "เกรด"]
data = [
    ("Alice", 22, 85.5, "B"),
    ("Bob", 20, 92.0, "A"),
    ("Charlie Longname", 23, 78.3, "C"),
]

# Header
print(f"{'ชื่อ':20} {'อายุ':5} {'คะแนน':8} {'เกรด':5}")
print("-" * 42)
for name, age, score, grade in data:
    print(f"{name:20} {age:5} {score:8.1f} {grade:5}")
```

### 2.3 str.format()

```python
# str.format() - Python 2 compatible style
print("Hello, {}!".format("Alice"))
print("{} + {} = {}".format(1, 2, 3))

# Positional
print("{0} loves {1} and {1} loves {0}".format("Alice", "Python"))

# Named
print("{name} is {age} years old".format(name="Bob", age=25))

# Format spec
print("{:.2f}".format(3.14159))    # 3.14
print("{:>10}".format("hello"))    # '     hello'
print("{:,}".format(1000000))      # 1,000,000

# Template strings
from string import Template
t = Template("Hello, $name! You are $age years old.")
print(t.substitute(name="Alice", age=30))
print(t.safe_substitute(name="Bob"))  # ไม่ error ถ้า key ขาด
```

---

## 3. String Methods ครบทุก Method

### 3.1 Case Methods

```python
s = "Hello World Python"

print(s.upper())       # HELLO WORLD PYTHON
print(s.lower())       # hello world python
print(s.title())       # Hello World Python
print(s.capitalize())  # Hello world python (ตัวแรกใหญ่ ที่เหลือเล็ก)
print(s.swapcase())    # hELLO wORLD pYTHON

# casefold() - aggressive lowercase (สำหรับ comparison)
print("Straße".lower())     # straße
print("Straße".casefold())  # strasse  ← เหมาะสำหรับ comparison

# ตรวจสอบ case
print("HELLO".isupper())    # True
print("hello".islower())    # True
print("Hello World".istitle())  # True
```

### 3.2 Search และ Check Methods

```python
s = "Hello World Python Programming"

# find() / rfind() - หาตำแหน่ง (คืน -1 ถ้าไม่พบ)
print(s.find("World"))    # 6
print(s.find("Java"))     # -1
print(s.rfind("o"))       # 20 (ตำแหน่งสุดท้าย)
print(s.find("o", 5))     # 7 (หาตั้งแต่ index 5)
print(s.find("o", 0, 10)) # 4 (หาในช่วง 0-10)

# index() / rindex() - หาตำแหน่ง (ValueError ถ้าไม่พบ)
print(s.index("World"))   # 6
try:
    s.index("Java")
except ValueError as e:
    print(f"ValueError: {e}")

# startswith() / endswith()
print(s.startswith("Hello"))    # True
print(s.startswith("World"))    # False
print(s.endswith("Programming")) # True
print(s.endswith("Python"))     # False

# startswith/endswith กับ tuple
print(s.startswith(("Hello", "Hi", "Hey")))  # True
print(s.endswith((".py", ".txt", ".md")))    # False

# count() - นับครั้งที่พบ
print(s.count("o"))     # 3
print(s.count("ll"))    # 1
print(s.count("l"))     # 3
print(s.count("o", 10)) # นับตั้งแต่ index 10

# in operator
print("Python" in s)      # True
print("Java" not in s)    # True

# Check methods
print("123".isdigit())        # True
print("123.45".isdigit())     # False
print("abc123".isalnum())     # True
print("hello".isalpha())      # True
print("   ".isspace())        # True
print("Hello World".istitle())  # True

# str.isdecimal(), str.isnumeric(), str.isdigit()
print("123".isdecimal())   # True
print("²".isdigit())       # True (superscript)
print("²".isdecimal())     # False
print("½".isnumeric())     # True
print("½".isdigit())       # False

# isidentifier() - valid Python identifier?
print("hello_world".isidentifier())  # True
print("123abc".isidentifier())       # False
print("class".isidentifier())        # True (แต่ reserved word!)
import keyword
print(keyword.iskeyword("class"))    # True
```

### 3.3 Strip Methods

```python
# strip() - ลบ whitespace (หรือ characters) ทั้งสองด้าน
s = "   Hello World   "
print(s.strip())           # 'Hello World'
print(s.lstrip())          # 'Hello World   '
print(s.rstrip())          # '   Hello World'

# ลบ specific characters
s2 = "***Hello World***"
print(s2.strip("*"))       # 'Hello World'
print(s2.lstrip("*"))      # 'Hello World***'
print(s2.rstrip("*"))      # '***Hello World'

# ลบ multiple chars (set ของ chars ไม่ใช่ substring)
s3 = "xyzHello Worldzyx"
print(s3.strip("xyz"))     # 'Hello World'

# removeprefix() / removesuffix() (Python 3.9+)
url = "https://www.example.com"
print(url.removeprefix("https://"))  # www.example.com
print(url.removesuffix(".com"))      # https://www.example

# ถ้า prefix/suffix ไม่พบ - คืนค่าเดิม
print("hello".removeprefix("bye"))   # hello (ไม่เปลี่ยน)

# Clean user input
def clean_input(text):
    return text.strip().lower()

user_input = "  HELLO WORLD  "
print(clean_input(user_input))  # hello world
```

### 3.4 Split Methods

```python
# split() - แยกด้วย separator
s = "apple,banana,cherry,date"
print(s.split(","))              # ['apple', 'banana', 'cherry', 'date']
print(s.split(",", maxsplit=2))  # ['apple', 'banana', 'cherry,date']

# split() ไม่มี argument - แยกด้วย whitespace
s2 = "Hello   World\tPython\nCode"
print(s2.split())  # ['Hello', 'World', 'Python', 'Code']

# rsplit() - แยกจากขวา
s3 = "a.b.c.d"
print(s3.rsplit(".", 1))   # ['a.b.c', 'd']
print(s3.rsplit(".", 2))   # ['a.b', 'c', 'd']

# splitlines() - แยกตาม line breaks
multiline = "line1\nline2\r\nline3\rline4"
print(multiline.splitlines())
# ['line1', 'line2', 'line3', 'line4']

# partition() - แยกเป็น 3 ส่วน (before, sep, after)
s4 = "user@example.com"
print(s4.partition("@"))    # ('user', '@', 'example.com')
print(s4.rpartition("."))   # ('user@example', '.', 'com')

# ตัวอย่าง: parse URL
url = "https://www.example.com:8080/path?query=value"
protocol, _, rest = url.partition("://")
host_part, _, path_part = rest.partition("/")
host, _, port = host_part.rpartition(":")
path, _, query = path_part.partition("?")

print(f"Protocol: {protocol}")  # https
print(f"Host: {host}")          # www.example.com
print(f"Port: {port}")          # 8080
print(f"Path: /{path}")         # /path
print(f"Query: {query}")        # query=value
```

### 3.5 Replace และ Join

```python
# replace() - แทนที่
s = "Hello World World World"
print(s.replace("World", "Python"))           # Hello Python Python Python
print(s.replace("World", "Python", count=1)) # Hello Python World World (count ของครั้ง)

# translate() + maketrans() - แทนที่หลาย chars พร้อมกัน
# สร้าง translation table
table = str.maketrans("aeiou", "AEIOU")  # แทนที่ vowels เป็นตัวใหญ่
print("hello world".translate(table))   # hEllO wOrld

# ลบ characters
table = str.maketrans("", "", "aeiou")   # ลบ vowels
print("hello world".translate(table))   # hll wrld

# complex translation
table = str.maketrans(
    "abc", "ABC",      # a→A, b→B, c→C
    "xyz",             # ลบ x, y, z
)
print("abcxyz123".translate(table))  # ABC123

# join() - เชื่อม strings
words = ["Hello", "World", "Python"]
print(" ".join(words))         # Hello World Python
print("-".join(words))         # Hello-World-Python
print("".join(words))          # HelloWorldPython
print(", ".join(words))        # Hello, World, Python

# join กับ generator expression
numbers = [1, 2, 3, 4, 5]
print(", ".join(str(n) for n in numbers))  # 1, 2, 3, 4, 5

# path joining
parts = ["home", "user", "documents", "file.txt"]
path = "/".join(parts)
print(f"/{path}")   # /home/user/documents/file.txt

# ⚠️ อย่าใช้ + ใน loop (ช้ามาก)
# ผิด
result = ""
for s in words:
    result += s  # สร้าง string ใหม่ทุกครั้ง O(n²)

# ถูก
result = "".join(words)  # O(n)
```

### 3.6 Padding Methods

```python
# center() / ljust() / rjust()
s = "hello"
print(s.center(20))         # '       hello        '
print(s.center(20, "*"))    # '*******hello********'
print(s.ljust(20))          # 'hello               '
print(s.ljust(20, "-"))     # 'hello---------------'
print(s.rjust(20))          # '               hello'
print(s.rjust(20, "0"))     # '000000000000000hello'

# zfill() - ใส่ 0 ด้านหน้าตัวเลข
print("42".zfill(5))      # 00042
print("-42".zfill(5))     # -0042
print("3.14".zfill(7))    # 0003.14

# ตัวอย่าง: สร้าง ID
for i in range(1, 6):
    product_id = f"PROD-{str(i).zfill(4)}"
    print(product_id)
# PROD-0001
# PROD-0002
# ...
```

### 3.7 Other Methods

```python
# encode() / decode()
s = "สวัสดี Python"
encoded = s.encode("utf-8")
print(encoded)              # b'\xe0\xb8\xaa\xe0\xb8\xa7...'
print(encoded.decode("utf-8"))  # สวัสดี Python

# expandtabs() - แปลง \t เป็น spaces
s = "col1\tcol2\tcol3"
print(s.expandtabs(10))  # col1      col2      col3

# isprintable()
print("hello".isprintable())    # True
print("hello\n".isprintable())  # False (มี \n)
print("🐍".isprintable())       # True

# maketrans() / translate() เพิ่มเติม
# ROT13 cipher
rot13 = str.maketrans(
    "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz",
    "NOPQRSTUVWXYZABCDEFGHIJKLMnopqrstuvwxyzabcdefghijklm"
)
message = "Hello World"
encrypted = message.translate(rot13)
decrypted = encrypted.translate(rot13)
print(f"Original:  {message}")
print(f"Encrypted: {encrypted}")
print(f"Decrypted: {decrypted}")
```

---

## 4. Regular Expressions (Regex)

```python
import re

# re.search() - หาตำแหน่งแรกที่ match
text = "My phone is 081-234-5678 and 082-345-6789"
m = re.search(r"\d{3}-\d{3}-\d{4}", text)
if m:
    print(f"พบ: {m.group()}")   # 081-234-5678
    print(f"ตำแหน่ง: {m.start()}-{m.end()}")

# re.findall() - หาทุกที่ที่ match
phones = re.findall(r"\d{3}-\d{3}-\d{4}", text)
print(phones)  # ['081-234-5678', '082-345-6789']

# re.finditer() - iterator ของ match objects
for m in re.finditer(r"\d{3}-\d{3}-\d{4}", text):
    print(f"  {m.group()} @ {m.start()}")

# re.match() - match ที่ต้น string เท่านั้น
print(re.match(r"My", text))     # match
print(re.match(r"phone", text))  # None (ไม่ได้อยู่ต้น)

# re.fullmatch() - match ทั้ง string
print(re.fullmatch(r"\d+", "12345"))  # match
print(re.fullmatch(r"\d+", "123abc")) # None

# re.sub() - แทนที่
result = re.sub(r"\d{3}-\d{3}-\d{4}", "[PHONE]", text)
print(result)  # My phone is [PHONE] and [PHONE]

# re.sub กับ function
def mask_phone(m):
    phone = m.group()
    return phone[:4] + "***" + phone[-4:]

masked = re.sub(r"\d{3}-\d{3}-\d{4}", mask_phone, text)
print(masked)  # My phone is 081-***5678 and 082-***6789

# re.split() - split ด้วย pattern
s = "one,two;three.four five"
parts = re.split(r"[,;.\s]+", s)
print(parts)  # ['one', 'two', 'three', 'four', 'five']
```

### 4.1 Patterns ทั่วไป

```python
import re

# Character classes
print(re.findall(r"\d", "abc123def456"))    # ['1','2','3','4','5','6']
print(re.findall(r"\D", "abc123"))          # ['a','b','c']
print(re.findall(r"\w", "hello world!"))    # ['h','e','l','l','o','w','o','r','l','d']
print(re.findall(r"\W", "hello world!"))    # [' ', '!']
print(re.findall(r"\s", "hello\tworld\n")) # ['\t', '\n']
print(re.findall(r"\S", "hello\tworld"))   # ['h','e','l','l','o','w','o','r','l','d']

# Quantifiers
text = "aabbbcccc"
print(re.findall(r"a+", text))    # ['aa']
print(re.findall(r"b*", text))    # ['', '', 'bbb', '', '', '', '', '']
print(re.findall(r"c?", text))    # ['', '', '', '', '', 'c', 'c', 'c', 'c', '']
print(re.findall(r"b{2}", text))  # ['bb']
print(re.findall(r"b{1,3}", text)) # ['bbb']

# Groups
dates = "Born: 1990-05-15, Died: 2023-12-31"
for m in re.finditer(r"(\d{4})-(\d{2})-(\d{2})", dates):
    year, month, day = m.groups()
    print(f"Date: {year}/{month}/{day}")

# Named groups
pattern = r"(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})"
for m in re.finditer(pattern, dates):
    print(f"Year: {m.group('year')}, Month: {m.group('month')}, Day: {m.group('day')}")

# Alternation
print(re.findall(r"cat|dog|fish", "I have a cat and a dog and a fish"))
# ['cat', 'dog', 'fish']

# Anchors
print(re.findall(r"^\w+", "Hello World\nPython Code", re.MULTILINE))
# ['Hello', 'Python']
print(re.findall(r"\w+$", "Hello World\nPython Code", re.MULTILINE))
# ['World', 'Code']

# Lookahead / Lookbehind
prices = "apple: $5.99, banana: $1.50, cherry: $12.99"
# หา numbers ที่ตามหลัง $
nums = re.findall(r"(?<=\$)\d+\.\d+", prices)
print(nums)  # ['5.99', '1.50', '12.99']
```

### 4.2 Compiled Patterns

```python
import re

# compile() pattern สำหรับใช้ซ้ำหลายครั้ง
EMAIL_PATTERN = re.compile(
    r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b'
)
PHONE_PATTERN = re.compile(r'\b\d{3}[-.\s]\d{3}[-.\s]\d{4}\b')
URL_PATTERN = re.compile(
    r'https?://(?:[-\w.]|(?:%[\da-fA-F]{2}))+'
)

texts = [
    "Contact alice@example.com or 081-234-5678",
    "Visit https://www.example.com for more info",
    "Email: bob@company.org, Phone: 082.345.6789",
]

for text in texts:
    emails = EMAIL_PATTERN.findall(text)
    phones = PHONE_PATTERN.findall(text)
    urls = URL_PATTERN.findall(text)
    
    if emails: print(f"Emails: {emails}")
    if phones: print(f"Phones: {phones}")
    if urls:   print(f"URLs: {urls}")
```

---

## 5. String Encoding

```python
# Encoding คือ การแปลง string เป็น bytes
# Decoding คือ การแปลง bytes เป็น string

# encode()
text = "สวัสดี Hello"
utf8_bytes = text.encode("utf-8")
utf16_bytes = text.encode("utf-16")
ascii_bytes = "Hello".encode("ascii")

print(f"UTF-8 ({len(utf8_bytes)} bytes): {utf8_bytes[:10]}...")
print(f"UTF-16 ({len(utf16_bytes)} bytes): {utf16_bytes[:10]}...")
print(f"ASCII ({len(ascii_bytes)} bytes): {ascii_bytes}")

# decode()
decoded_utf8 = utf8_bytes.decode("utf-8")
decoded_utf16 = utf16_bytes.decode("utf-16")

print(decoded_utf8)   # สวัสดี Hello
print(decoded_utf16)  # สวัสดี Hello

# Handling errors
bad_bytes = b"\xff\xfe\x00Hello"
try:
    text = bad_bytes.decode("ascii")
except UnicodeDecodeError as e:
    print(f"Error: {e}")

# errors parameter
text = bad_bytes.decode("ascii", errors="ignore")   # ข้ามตัวที่ decode ไม่ได้
print(f"ignore: {text!r}")

text = bad_bytes.decode("ascii", errors="replace")  # แทนที่ด้วย ?
print(f"replace: {text!r}")

# bytes operations
data = b"Hello, World!"
print(data[0])           # 72
print(data[7:12])        # b'World'
print(data.upper())      # b'HELLO, WORLD!'
print(data.replace(b"World", b"Python"))  # b'Hello, Python!'

# bytearray - mutable bytes
ba = bytearray(b"Hello")
ba[0] = 104  # ASCII 'h'
print(ba)    # bytearray(b'hello')
ba.extend(b" World")
print(ba)    # bytearray(b'hello World')
```

---

## 6. ตัวอย่างโปรแกรมจริง: Text Processor

```python
"""
Text Processor สำหรับวิเคราะห์และแปลง text
"""
import re
from collections import Counter
from typing import List, Dict, Optional, Tuple

class TextProcessor:
    """คลาสสำหรับประมวลผล text"""
    
    # Patterns
    EMAIL_RE = re.compile(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}\b')
    PHONE_RE = re.compile(r'\b(?:\+66|0)[\d\s-]{8,11}\b')
    URL_RE = re.compile(r'https?://\S+')
    HASHTAG_RE = re.compile(r'#\w+')
    MENTION_RE = re.compile(r'@\w+')
    
    def __init__(self, text: str):
        self.original = text
        self.text = text
    
    def clean(self, remove_urls=True, remove_html=True, normalize_whitespace=True):
        """ทำความสะอาด text"""
        t = self.text
        
        if remove_html:
            t = re.sub(r'<[^>]+>', '', t)
        
        if remove_urls:
            t = self.URL_RE.sub('', t)
        
        if normalize_whitespace:
            t = re.sub(r'\s+', ' ', t).strip()
        
        self.text = t
        return self
    
    def extract_entities(self) -> Dict[str, List[str]]:
        """ดึง entities จาก text"""
        return {
            "emails": self.EMAIL_RE.findall(self.text),
            "phones": self.PHONE_RE.findall(self.text),
            "urls": self.URL_RE.findall(self.text),
            "hashtags": self.HASHTAG_RE.findall(self.text),
            "mentions": self.MENTION_RE.findall(self.text),
        }
    
    def word_frequency(self, top_n: int = 10, min_length: int = 3) -> List[Tuple[str, int]]:
        """นับความถี่ของคำ"""
        words = re.findall(r'\b[a-zA-Z]+\b', self.text.lower())
        words = [w for w in words if len(w) >= min_length]
        return Counter(words).most_common(top_n)
    
    def sentence_count(self) -> int:
        """นับจำนวนประโยค"""
        sentences = re.split(r'[.!?]+', self.text)
        return len([s for s in sentences if s.strip()])
    
    def replace_sensitive(self, replacement: str = "***") -> "TextProcessor":
        """แทนที่ข้อมูล sensitive ด้วย placeholder"""
        t = self.text
        t = self.EMAIL_RE.sub(f"[EMAIL]", t)
        t = self.PHONE_RE.sub(f"[PHONE]", t)
        self.text = t
        return self
    
    def highlight(self, keywords: List[str], marker: str = "**") -> str:
        """เน้น keywords"""
        result = self.text
        for kw in keywords:
            pattern = re.compile(re.escape(kw), re.IGNORECASE)
            result = pattern.sub(f"{marker}{kw}{marker}", result)
        return result
    
    def stats(self) -> Dict:
        """สถิติของ text"""
        words = self.text.split()
        chars = len(self.text)
        
        return {
            "characters": chars,
            "characters_no_space": len(self.text.replace(" ", "")),
            "words": len(words),
            "unique_words": len(set(w.lower() for w in words)),
            "sentences": self.sentence_count(),
            "avg_word_length": sum(len(w) for w in words) / len(words) if words else 0,
            "longest_word": max(words, key=len) if words else "",
        }

# ทดสอบ
article = """
Python is an amazing programming language! Contact our team at support@example.com or 
call 081-234-5678. Visit https://www.python.org for more info.
#Python #Programming @pythondev

Python's design philosophy emphasizes code readability. The language uses 
significant whitespace. Python was created by Guido van Rossum and first 
released in 1991. It has become one of the most popular programming languages.
Email sales@company.com for pricing.
"""

processor = TextProcessor(article)
entities = processor.extract_entities()
print("=== Entities ===")
for entity_type, items in entities.items():
    if items:
        print(f"  {entity_type}: {items}")

stats = processor.stats()
print("\n=== Statistics ===")
for key, val in stats.items():
    if isinstance(val, float):
        print(f"  {key}: {val:.2f}")
    else:
        print(f"  {key}: {val}")

print("\n=== Top 5 Words ===")
for word, count in processor.word_frequency(5):
    print(f"  {word}: {count}")

print("\n=== After Masking Sensitive Info ===")
masked = TextProcessor(article).replace_sensitive()
print(masked.text[:200])
```

---

## 7. Exercises

### Exercise 1: Password Validator

```python
"""
สร้าง Password Validator ที่ตรวจสอบ:
1. ความยาวอย่างน้อย 8 ตัวอักษร
2. มีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว
3. มีตัวพิมพ์เล็กอย่างน้อย 1 ตัว
4. มีตัวเลขอย่างน้อย 1 ตัว
5. มี special character อย่างน้อย 1 ตัว
6. คำนวณ strength score
"""
import re

def validate_password(password: str) -> dict:
    checks = {
        "min_length": len(password) >= 8,
        "has_uppercase": bool(re.search(r'[A-Z]', password)),
        "has_lowercase": bool(re.search(r'[a-z]', password)),
        "has_digit": bool(re.search(r'\d', password)),
        "has_special": bool(re.search(r'[!@#$%^&*(),.?":{}|<>]', password)),
        "no_spaces": ' ' not in password,
        "long_enough": len(password) >= 12,  # bonus
    }
    
    passed = sum(1 for v in checks.values() if v)
    total = len(checks)
    
    # Strength score
    if passed <= 3:
        strength = "WEAK"
    elif passed <= 5:
        strength = "MEDIUM"
    elif passed <= 6:
        strength = "STRONG"
    else:
        strength = "VERY STRONG"
    
    return {
        "valid": all(v for k, v in checks.items() if k != "long_enough"),
        "strength": strength,
        "score": f"{passed}/{total}",
        "checks": checks,
    }

# ทดสอบ
passwords = [
    "abc",
    "password",
    "Password1",
    "Password1!",
    "MyStr0ng@Pass!",
]

for pwd in passwords:
    result = validate_password(pwd)
    status = "✓ VALID" if result["valid"] else "✗ INVALID"
    print(f"\n[{status}] '{pwd}' - {result['strength']} ({result['score']})")
    for check, passed in result["checks"].items():
        icon = "  ✓" if passed else "  ✗"
        print(f"{icon} {check}")
```

### Exercise 2: Template Engine

```python
"""
สร้าง Simple Template Engine:
1. แทนที่ {{variable}} ด้วยค่าจริง
2. รองรับ {{if condition}}...{{endif}}
3. รองรับ {{for item in list}}...{{endfor}}
"""
import re
from typing import Any, Dict

def render_template(template: str, context: Dict[str, Any]) -> str:
    """Simple template renderer"""
    result = template
    
    # แทนที่ {{ variable }}
    def replace_var(m):
        key = m.group(1).strip()
        # รองรับ dot notation
        value = context
        for part in key.split("."):
            if isinstance(value, dict):
                value = value.get(part, f"{{{{ {key} }}}}")
            elif hasattr(value, part):
                value = getattr(value, part)
            else:
                return m.group(0)  # คืนค่าเดิมถ้าหาไม่พบ
        return str(value)
    
    result = re.sub(r'\{\{\s*(\w+(?:\.\w+)*)\s*\}\}', replace_var, result)
    
    # แทนที่ {% for ... %} ... {% endfor %}
    for_pattern = re.compile(
        r'\{%\s*for\s+(\w+)\s+in\s+(\w+)\s*%\}(.*?)\{%\s*endfor\s*%\}',
        re.DOTALL
    )
    
    def replace_for(m):
        var_name, list_name, body = m.group(1), m.group(2), m.group(3)
        items = context.get(list_name, [])
        parts = []
        for item in items:
            local_ctx = {**context, var_name: item}
            parts.append(render_template(body, local_ctx))
        return "".join(parts)
    
    result = for_pattern.sub(replace_for, result)
    
    return result

# ทดสอบ
template = """
สวัสดี {{ name }}!

รายงานประจำ{{ period }}:
{% for item in items %}  - {{ item }}
{% endfor %}
สรุป: มีทั้งหมด {{ count }} รายการ
ติดต่อ: {{ contact.email }}
"""

context = {
    "name": "Alice",
    "period": "สัปดาห์",
    "items": ["งานที่ 1 เสร็จแล้ว", "งานที่ 2 กำลังทำ", "งานที่ 3 รอการตรวจ"],
    "count": "3",
    "contact": {"email": "manager@company.com", "phone": "02-xxx-xxxx"},
}

print(render_template(template, context))
```

### Exercise 3: Log Parser

```python
"""
Parse log files:
1. ดึงข้อมูล: timestamp, level, service, message
2. กรองตาม level
3. สถิติ error rate
"""
import re
from datetime import datetime
from collections import Counter
from typing import List, Dict, Optional

LOG_PATTERN = re.compile(
    r'(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})'  # timestamp
    r'\s+\[(\w+)\]'                              # level
    r'\s+(\w+(?:\.\w+)*)'                        # service
    r'\s+-\s+(.+)'                               # message
)

def parse_log_line(line: str) -> Optional[Dict]:
    m = LOG_PATTERN.match(line.strip())
    if not m:
        return None
    return {
        "timestamp": m.group(1),
        "level": m.group(2),
        "service": m.group(3),
        "message": m.group(4),
    }

def analyze_logs(log_text: str) -> Dict:
    lines = log_text.strip().splitlines()
    entries = [parse_log_line(line) for line in lines]
    entries = [e for e in entries if e]  # ลบ None
    
    level_counts = Counter(e["level"] for e in entries)
    service_errors = Counter(
        e["service"] for e in entries if e["level"] == "ERROR"
    )
    
    return {
        "total": len(entries),
        "by_level": dict(level_counts),
        "error_rate": level_counts.get("ERROR", 0) / len(entries) if entries else 0,
        "top_error_services": service_errors.most_common(3),
        "errors": [e for e in entries if e["level"] == "ERROR"],
    }

# ทดสอบ
sample_logs = """
2024-01-15 08:30:01 [INFO] app.server - Server started on port 8080
2024-01-15 08:30:05 [INFO] app.database - Connected to database
2024-01-15 08:31:00 [INFO] app.server - Request GET /api/users
2024-01-15 08:31:01 [DEBUG] app.database - Query executed in 45ms
2024-01-15 08:31:15 [WARN] app.server - Request timeout for /api/orders
2024-01-15 08:32:00 [ERROR] app.database - Connection pool exhausted
2024-01-15 08:32:01 [ERROR] app.server - Failed to process request: DB unavailable
2024-01-15 08:32:30 [INFO] app.cache - Cache miss for key user:123
2024-01-15 08:33:00 [ERROR] app.database - Query failed: timeout after 5000ms
2024-01-15 08:33:15 [INFO] app.server - Health check OK
"""

result = analyze_logs(sample_logs)
print(f"Log entries: {result['total']}")
print(f"By level: {result['by_level']}")
print(f"Error rate: {result['error_rate']:.1%}")
print(f"Top error services: {result['top_error_services']}")
print("\nErrors:")
for err in result["errors"]:
    print(f"  [{err['timestamp']}] {err['service']}: {err['message']}")
```

---

## 8. สรุป Part 011

### สิ่งที่เรียนรู้:

✅ **f-strings ขั้นสูง** - format spec, debug mode `=`, expressions  
✅ **Format Spec** - width, alignment, fill, precision, type  
✅ **String Methods** - case, search, strip, split, replace, join  
✅ **Regex** - `re.search()`, `re.findall()`, `re.sub()`, patterns  
✅ **Raw Strings** - `r"..."` สำหรับ regex และ paths  
✅ **Encoding** - `.encode()`, `.decode()`, UTF-8, ASCII  
✅ **Bytes** - `b"..."`, bytearray  
✅ **Multiline Strings** - `"""..."""`, `'''...'''`  

### Quick Reference:

```python
# Format spec
f"{value:.2f}"      # 2 decimal places
f"{value:,}"        # thousands separator
f"{value:>10}"      # right-align width 10
f"{value:*^10}"     # center with * fill
f"{value:#x}"       # hex with prefix
f"{value:.1%}"      # percentage

# Common methods
s.upper() / s.lower() / s.title()
s.strip() / s.lstrip() / s.rstrip()
s.split(",") / ", ".join(list)
s.replace("old", "new")
s.startswith("x") / s.endswith("y")
s.find("x") / s.count("x")
s.encode("utf-8") / b.decode("utf-8")

# Regex
import re
re.findall(r'\d+', text)
re.sub(r'pattern', 'replacement', text)
re.search(r'pattern', text)
re.compile(r'pattern')  # compiled pattern
```

---

## ➡️ ถัดไป: Part 012 - File I/O

ใน Part ถัดไป เราจะเรียนรู้:
- อ่าน/เขียนไฟล์ text และ binary
- Context manager (with statement)
- CSV และ JSON files
- pathlib สำหรับ file/directory operations

---

*Part 011/100+ | Python Course - Beginner to World-Class*
