# Part 009: Dictionaries
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- สร้างและใช้งาน Dictionary ได้อย่างคล่องแคล่ว
- ทำ CRUD operations บน Dictionary ได้
- ใช้ Dictionary methods ต่างๆ ได้ (keys, values, items, get, update, pop)
- เขียน Dict Comprehensions ได้
- ใช้ defaultdict, Counter, OrderedDict ได้
- จัดการ Nested Dictionaries ได้

---

## 1. Dictionary พื้นฐาน

Dictionary คือ data structure แบบ key-value pairs ที่:
- **Mutable** - แก้ไขได้
- **Ordered** - รักษาลำดับ insertion (Python 3.7+)
- **No duplicate keys** - key ซ้ำกันไม่ได้
- **Fast lookup** - ค้นหาด้วย O(1)

```python
# ===== การสร้าง Dictionary =====

# วิธีที่ 1: ใช้ curly braces {}
empty_dict = {}
person = {
    "name": "สมชาย",
    "age": 30,
    "city": "กรุงเทพ"
}

# วิธีที่ 2: ใช้ dict() constructor
person2 = dict(name="สมหญิง", age=25, city="เชียงใหม่")

# วิธีที่ 3: จาก list of tuples
pairs = [("a", 1), ("b", 2), ("c", 3)]
from_tuples = dict(pairs)
print(from_tuples)  # {'a': 1, 'b': 2, 'c': 3}

# วิธีที่ 4: dict.fromkeys()
keys = ["x", "y", "z"]
zeros = dict.fromkeys(keys, 0)
print(zeros)  # {'x': 0, 'y': 0, 'z': 0}

# Key types - key ต้องเป็น immutable
valid_keys = {
    42: "integer key",
    3.14: "float key",
    "text": "string key",
    (1, 2): "tuple key",
    True: "boolean key",
}

# ⚠️ list ใช้เป็น key ไม่ได้ (mutable)
# invalid = {[1, 2]: "list key"}  # TypeError!

print(f"จำนวน keys: {len(person)}")   # 3
print(f"ชนิด: {type(person)}")        # <class 'dict'>
```

## 2. การเข้าถึงข้อมูล (Read)

```python
student = {
    "name": "นักเรียน A",
    "grade": 10,
    "scores": [85, 90, 78, 92],
    "passed": True
}

# วิธีที่ 1: ใช้ [] - ถ้า key ไม่มีจะ raise KeyError
name = student["name"]
print(name)  # นักเรียน A

# วิธีที่ 2: ใช้ .get() - ถ้า key ไม่มีจะ return None หรือ default
grade = student.get("grade")           # 10
subject = student.get("subject")       # None (ไม่ error!)
subject = student.get("subject", "ไม่มีวิชา")  # ค่า default

print(f"เกรด: {grade}")
print(f"วิชา: {subject}")  # ไม่มีวิชา

# ตรวจสอบว่า key มีอยู่ไหม
if "name" in student:
    print(f"ชื่อ: {student['name']}")

if "email" not in student:
    print("ไม่มีข้อมูลอีเมล")

# เข้าถึง nested values
scores = student["scores"]
first_score = student["scores"][0]
print(f"คะแนนแรก: {first_score}")  # 85
```

## 3. การเพิ่ม/แก้ไขข้อมูล (Create/Update)

```python
profile = {"username": "user001"}

# เพิ่ม key ใหม่
profile["email"] = "user@example.com"
profile["age"] = 25

# แก้ไข value ที่มีอยู่
profile["age"] = 26  # อัพเดต age

# ใช้ .update() - อัพเดตหลาย key พร้อมกัน
profile.update({
    "city": "กรุงเทพ",
    "phone": "0812345678"
})

# update ด้วย keyword arguments
profile.update(age=27, active=True)

# setdefault() - เพิ่มเฉพาะถ้า key ยังไม่มี
profile.setdefault("country", "ไทย")   # เพิ่ม country
profile.setdefault("email", "other@example.com")  # ไม่เปลี่ยน email ที่มีอยู่แล้ว

print(profile)

# Merge dictionaries (Python 3.9+)
base = {"a": 1, "b": 2}
extra = {"b": 99, "c": 3}

# วิธี 1: | operator
merged = base | extra
print(merged)  # {'a': 1, 'b': 99, 'c': 3}

# วิธี 2: **unpacking (ทุก version)
merged2 = {**base, **extra}
print(merged2)  # {'a': 1, 'b': 99, 'c': 3}

# วิธี 3: .update()
base.update(extra)  # แก้ไข base โดยตรง
print(base)
```

## 4. การลบข้อมูล (Delete)

```python
inventory = {
    "apple": 50,
    "banana": 30,
    "cherry": 20,
    "date": 10,
    "elderberry": 5
}

# วิธีที่ 1: del
del inventory["date"]
print("หลัง del:", inventory)

# วิธีที่ 2: .pop() - ลบและ return value
removed = inventory.pop("banana")
print(f"ลบ banana ออก, มีจำนวน: {removed}")

# pop กับ default value
val = inventory.pop("fig", 0)  # ถ้าไม่มีจะ return 0 แทน error
print(f"fig: {val}")  # 0

# วิธีที่ 3: .popitem() - ลบและ return คู่สุดท้าย
last_item = inventory.popitem()
print(f"ลบรายการสุดท้าย: {last_item}")

# วิธีที่ 4: .clear() - ล้างทั้งหมด
backup = inventory.copy()
inventory.clear()
print(f"หลัง clear: {inventory}")  # {}

# copy กลับมา
inventory = backup.copy()
print("restore:", inventory)
```

## 5. Dictionary Methods สำคัญ

```python
product = {
    "id": "P001",
    "name": "Python Book",
    "price": 599,
    "stock": 100,
    "category": "books"
}

# .keys() - ได้ dict_keys object
keys = product.keys()
print(f"Keys: {list(keys)}")

# .values() - ได้ dict_values object
values = product.values()
print(f"Values: {list(values)}")

# .items() - ได้ dict_items object (tuples)
items = product.items()
print(f"Items: {list(items)}")

# วนลูปผ่าน keys
print("\nวนลูปผ่าน keys:")
for key in product:
    print(f"  {key}")

# วนลูปผ่าน values
print("\nวนลูปผ่าน values:")
for value in product.values():
    print(f"  {value}")

# วนลูปผ่าน key-value pairs
print("\nวนลูปผ่าน items:")
for key, value in product.items():
    print(f"  {key}: {value}")

# len() - จำนวน key-value pairs
print(f"\nจำนวน fields: {len(product)}")

# in operator
print(f"มี 'price'? {'price' in product}")   # True
print(f"มี 'discount'? {'discount' in product}")  # False

# ตรวจสอบ value
print(f"มีค่า 599? {599 in product.values()}")  # True
```

## 6. Dict Comprehensions

```python
# รูปแบบพื้นฐาน: {key_expr: value_expr for item in iterable}

# ===== ตัวอย่างที่ 1: แปลง list เป็น dict =====
words = ["apple", "banana", "cherry", "date"]
word_lengths = {word: len(word) for word in words}
print(word_lengths)
# {'apple': 5, 'banana': 6, 'cherry': 6, 'date': 4}

# ===== ตัวอย่างที่ 2: กรองข้อมูล =====
prices = {"apple": 25, "banana": 15, "cherry": 80, "grape": 120}
expensive = {item: price for item, price in prices.items() if price > 50}
print(expensive)
# {'cherry': 80, 'grape': 120}

# ===== ตัวอย่างที่ 3: แปลง values =====
celsius = {"กรุงเทพ": 35, "เชียงใหม่": 28, "ภูเก็ต": 33}
fahrenheit = {city: (temp * 9/5) + 32 for city, temp in celsius.items()}
print(fahrenheit)

# ===== ตัวอย่างที่ 4: swap keys and values =====
original = {"a": 1, "b": 2, "c": 3}
swapped = {v: k for k, v in original.items()}
print(swapped)  # {1: 'a', 2: 'b', 3: 'c'}

# ===== ตัวอย่างที่ 5: สร้าง lookup table =====
students = ["สมชาย", "สมหญิง", "สมศักดิ์", "สมใจ"]
student_ids = {name: f"STU{i+1:03d}" for i, name in enumerate(students)}
print(student_ids)
# {'สมชาย': 'STU001', 'สมหญิง': 'STU002', ...}

# ===== ตัวอย่างที่ 6: nested comprehension =====
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
flat = {(i, j): matrix[i][j] for i in range(3) for j in range(3)}
print(flat[(1, 2)])  # 6
```

## 7. defaultdict

```python
from collections import defaultdict

# ปัญหาปกติ: KeyError เมื่อ key ไม่มี
# normal_dict = {}
# normal_dict["key"].append(1)  # KeyError!

# วิธีแก้ 1: ตรวจสอบก่อน (verbose)
normal_dict = {}
if "fruits" not in normal_dict:
    normal_dict["fruits"] = []
normal_dict["fruits"].append("apple")

# วิธีแก้ 2: ใช้ defaultdict (clean!)
dd_list = defaultdict(list)
dd_list["fruits"].append("apple")
dd_list["fruits"].append("banana")
dd_list["vegetables"].append("carrot")
print(dict(dd_list))
# {'fruits': ['apple', 'banana'], 'vegetables': ['carrot']}

# defaultdict(int) - นับจำนวน
dd_int = defaultdict(int)
words = ["python", "java", "python", "c++", "python", "java"]
for word in words:
    dd_int[word] += 1  # ไม่ต้องกังวล KeyError
print(dict(dd_int))
# {'python': 3, 'java': 2, 'c++': 1}

# defaultdict(set) - เก็บ unique values
dd_set = defaultdict(set)
purchases = [
    ("สมชาย", "หนังสือ"),
    ("สมหญิง", "ปากกา"),
    ("สมชาย", "ยางลบ"),
    ("สมชาย", "หนังสือ"),  # ซ้ำ - จะไม่ถูกเพิ่ม
]
for customer, item in purchases:
    dd_set[customer].add(item)
print(dict(dd_set))
# {'สมชาย': {'หนังสือ', 'ยางลบ'}, 'สมหญิง': {'ปากกา'}}

# กำหนด default factory เอง
def create_default():
    return {"count": 0, "total": 0}

dd_custom = defaultdict(create_default)
dd_custom["product_A"]["count"] += 1
dd_custom["product_A"]["total"] += 500
print(dict(dd_custom))
```

## 8. Counter

```python
from collections import Counter

# นับความถี่ของ elements

# นับตัวอักษร
text = "mississippi"
char_count = Counter(text)
print(char_count)
# Counter({'i': 4, 's': 4, 'p': 2, 'm': 1})

# นับคำ
words = ["cat", "dog", "cat", "bird", "dog", "cat", "fish"]
word_count = Counter(words)
print(word_count)
# Counter({'cat': 3, 'dog': 2, 'bird': 1, 'fish': 1})

# most_common() - หา n อันดับแรก
print(word_count.most_common(2))
# [('cat', 3), ('dog', 2)]

# Counter arithmetic
counter1 = Counter({"a": 3, "b": 2, "c": 1})
counter2 = Counter({"a": 1, "b": 4, "d": 2})

# บวก
print(counter1 + counter2)  # Counter({'b': 6, 'a': 4, 'd': 2, 'c': 1})

# ลบ (เฉพาะค่าบวก)
print(counter1 - counter2)  # Counter({'a': 2, 'c': 1})

# intersection (min)
print(counter1 & counter2)  # Counter({'a': 1, 'b': 2})

# union (max)
print(counter1 | counter2)  # Counter({'b': 4, 'a': 3, 'd': 2, 'c': 1})

# ตัวอย่างใช้งานจริง: วิเคราะห์ log
logs = [
    "ERROR: connection failed",
    "INFO: user logged in",
    "ERROR: timeout",
    "WARNING: disk space low",
    "ERROR: connection failed",
    "INFO: file saved",
    "ERROR: connection failed",
]

log_levels = Counter(log.split(":")[0] for log in logs)
print("\nสรุป log levels:")
for level, count in log_levels.most_common():
    print(f"  {level}: {count} ครั้ง")
```

## 9. OrderedDict

```python
from collections import OrderedDict

# ใน Python 3.7+ dict ธรรมดาก็รักษาลำดับแล้ว
# แต่ OrderedDict มีความสามารถพิเศษ

od = OrderedDict()
od["first"] = 1
od["second"] = 2
od["third"] = 3

print(list(od.keys()))  # ['first', 'second', 'third']

# move_to_end() - ย้าย element ไปหน้า/หลัง
od.move_to_end("first")         # ย้ายไปท้าย
od.move_to_end("third", last=False)  # ย้ายไปหน้า

print(list(od.keys()))  # ['third', 'second', 'first']

# popitem() กับ OrderedDict
last = od.popitem(last=True)    # ลบ item สุดท้าย
first = od.popitem(last=False)  # ลบ item แรก

print(f"ลบหลัง: {last}")
print(f"ลบหน้า: {first}")

# เปรียบเทียบลำดับ
od1 = OrderedDict([("a", 1), ("b", 2)])
od2 = OrderedDict([("b", 2), ("a", 1)])
print(f"od1 == od2: {od1 == od2}")  # False (ลำดับต่างกัน)

d1 = {"a": 1, "b": 2}
d2 = {"b": 2, "a": 1}
print(f"d1 == d2: {d1 == d2}")  # True (dict ธรรมดาไม่สนลำดับ)
```

## 10. Nested Dictionaries

```python
# ===== โครงสร้าง Nested Dict =====
company = {
    "name": "Python Corp",
    "departments": {
        "engineering": {
            "head": "สมชาย วิศวกร",
            "employees": 50,
            "budget": 5000000
        },
        "marketing": {
            "head": "สมหญิง มาร์เก็ต",
            "employees": 20,
            "budget": 2000000
        },
        "hr": {
            "head": "สมศักดิ์ ฝ่ายคน",
            "employees": 10,
            "budget": 1000000
        }
    },
    "founded": 2010
}

# เข้าถึง nested values
eng_head = company["departments"]["engineering"]["head"]
print(f"หัวหน้า Engineering: {eng_head}")

# แก้ไข nested value
company["departments"]["engineering"]["budget"] += 500000
print(f"Budget ใหม่: {company['departments']['engineering']['budget']}")

# วนลูปผ่าน nested dict
print("\nรายละเอียดแผนก:")
for dept_name, dept_info in company["departments"].items():
    print(f"\n  {dept_name.upper()}:")
    for key, value in dept_info.items():
        print(f"    {key}: {value}")

# ===== เพิ่ม nested structure ใหม่ =====
company["departments"]["sales"] = {
    "head": "สมใจ ขาย",
    "employees": 30,
    "budget": 3000000
}

# ===== ใช้ get() กับ nested =====
# ปัญหา: ถ้า key ไม่มีจะ error
# deep_val = company["departments"]["finance"]["head"]  # KeyError!

# วิธีแก้ปลอดภัย
finance = company.get("departments", {}).get("finance", {})
finance_head = finance.get("head", "ไม่มีข้อมูล")
print(f"\nหัวหน้า Finance: {finance_head}")

# ===== ตัวอย่าง: ระบบนักศึกษา =====
university = {
    "students": {
        "STU001": {
            "name": "สมชาย ใจดี",
            "gpa": 3.8,
            "courses": ["CS101", "CS102", "MATH101"]
        },
        "STU002": {
            "name": "สมหญิง สวยงาม",
            "gpa": 3.5,
            "courses": ["CS101", "PHYS101"]
        }
    }
}

# หานักศึกษา GPA สูงสุด
best_student = max(
    university["students"].items(),
    key=lambda x: x[1]["gpa"]
)
print(f"\nนักศึกษา GPA สูงสุด: {best_student[1]['name']} (GPA: {best_student[1]['gpa']})")
```

## 11. เทคนิคขั้นสูง

```python
# ===== Inverting a Dictionary =====
phone_book = {
    "สมชาย": "081-234-5678",
    "สมหญิง": "082-345-6789",
    "สมศักดิ์": "083-456-7890"
}

# swap keys and values
reverse_lookup = {phone: name for name, phone in phone_book.items()}
print(reverse_lookup.get("081-234-5678"))  # สมชาย

# ===== Grouping Data =====
students = [
    {"name": "Alice", "grade": "A"},
    {"name": "Bob", "grade": "B"},
    {"name": "Charlie", "grade": "A"},
    {"name": "Diana", "grade": "C"},
    {"name": "Eve", "grade": "B"},
]

from collections import defaultdict

grouped = defaultdict(list)
for student in students:
    grouped[student["grade"]].append(student["name"])

print("\nนักเรียนแยกตามเกรด:")
for grade, names in sorted(grouped.items()):
    print(f"  เกรด {grade}: {', '.join(names)}")

# ===== Dictionary as a Switch/Case =====
def calculate(operation, a, b):
    operations = {
        "add": lambda x, y: x + y,
        "sub": lambda x, y: x - y,
        "mul": lambda x, y: x * y,
        "div": lambda x, y: x / y if y != 0 else "หารด้วยศูนย์ไม่ได้"
    }
    func = operations.get(operation)
    if func:
        return func(a, b)
    return f"ไม่รู้จัก operation: {operation}"

print(calculate("add", 10, 5))   # 15
print(calculate("mul", 3, 4))    # 12
print(calculate("div", 10, 0))   # หารด้วยศูนย์ไม่ได้
print(calculate("mod", 10, 3))   # ไม่รู้จัก operation: mod

# ===== Merging Nested Dicts =====
def deep_merge(base: dict, override: dict) -> dict:
    """Merge two dicts recursively"""
    result = base.copy()
    for key, value in override.items():
        if key in result and isinstance(result[key], dict) and isinstance(value, dict):
            result[key] = deep_merge(result[key], value)
        else:
            result[key] = value
    return result

config_default = {
    "database": {"host": "localhost", "port": 5432},
    "cache": {"ttl": 300}
}
config_prod = {
    "database": {"host": "prod-db.example.com"},
    "debug": False
}

merged_config = deep_merge(config_default, config_prod)
print("\nConfig merged:")
import json
print(json.dumps(merged_config, indent=2, ensure_ascii=False))
```

## 12. ตัวอย่างโปรเจกต์: ระบบจัดการสินค้า

```python
"""
ระบบจัดการสินค้าคลังสินค้า (Inventory Management)
"""

class Inventory:
    def __init__(self):
        self._items = {}  # {product_id: {name, price, quantity, category}}
    
    def add_item(self, product_id: str, name: str, price: float, 
                 quantity: int, category: str) -> None:
        """เพิ่มสินค้าใหม่"""
        if product_id in self._items:
            print(f"⚠️  สินค้า {product_id} มีอยู่แล้ว ใช้ update_stock แทน")
            return
        
        self._items[product_id] = {
            "name": name,
            "price": price,
            "quantity": quantity,
            "category": category
        }
        print(f"✅ เพิ่ม {name} แล้ว")
    
    def get_item(self, product_id: str) -> dict:
        """ดูข้อมูลสินค้า"""
        return self._items.get(product_id, {})
    
    def update_stock(self, product_id: str, quantity_change: int) -> bool:
        """อัพเดตจำนวนสินค้า (+เพิ่ม, -ลด)"""
        if product_id not in self._items:
            print(f"❌ ไม่พบสินค้า {product_id}")
            return False
        
        new_qty = self._items[product_id]["quantity"] + quantity_change
        if new_qty < 0:
            print(f"❌ สินค้าไม่พอ (มี {self._items[product_id]['quantity']} ชิ้น)")
            return False
        
        self._items[product_id]["quantity"] = new_qty
        return True
    
    def get_by_category(self, category: str) -> list:
        """หาสินค้าตาม category"""
        return [
            {**{"id": pid}, **info}
            for pid, info in self._items.items()
            if info["category"] == category
        ]
    
    def get_low_stock(self, threshold: int = 10) -> list:
        """หาสินค้าที่เหลือน้อย"""
        return [
            {"id": pid, "name": info["name"], "quantity": info["quantity"]}
            for pid, info in self._items.items()
            if info["quantity"] <= threshold
        ]
    
    def get_total_value(self) -> float:
        """คำนวณมูลค่าสินค้าทั้งหมด"""
        return sum(
            item["price"] * item["quantity"]
            for item in self._items.values()
        )
    
    def summary(self) -> dict:
        """สรุปข้อมูลคลังสินค้า"""
        from collections import Counter
        categories = Counter(item["category"] for item in self._items.values())
        return {
            "total_products": len(self._items),
            "total_value": self.get_total_value(),
            "categories": dict(categories),
            "low_stock_items": len(self.get_low_stock())
        }


# ===== ทดสอบระบบ =====
inv = Inventory()

# เพิ่มสินค้า
inv.add_item("P001", "Python Book", 599, 50, "books")
inv.add_item("P002", "USB Hub", 890, 30, "electronics")
inv.add_item("P003", "Mechanical Keyboard", 3500, 8, "electronics")
inv.add_item("P004", "Notebook", 89, 200, "stationery")
inv.add_item("P005", "Flask Book", 650, 5, "books")

# ดูข้อมูลสินค้า
item = inv.get_item("P001")
print(f"\nสินค้า P001: {item}")

# อัพเดตสต็อก
inv.update_stock("P001", -5)   # ขายไป 5 เล่ม
inv.update_stock("P002", 20)   # รับเข้า 20 ชิ้น

# หาสินค้าตาม category
books = inv.get_by_category("books")
print(f"\nหนังสือทั้งหมด: {len(books)} รายการ")
for book in books:
    print(f"  - {book['name']}: {book['quantity']} เล่ม @ {book['price']} บาท")

# สินค้าที่เหลือน้อย
low = inv.get_low_stock(threshold=10)
print(f"\nสินค้าเหลือน้อย (≤ 10 ชิ้น):")
for item in low:
    print(f"  ⚠️  {item['name']}: {item['quantity']} ชิ้น")

# สรุป
summary = inv.summary()
print(f"\n===== สรุปคลังสินค้า =====")
print(f"สินค้าทั้งหมด: {summary['total_products']} รายการ")
print(f"มูลค่ารวม: {summary['total_value']:,.2f} บาท")
print(f"หมวดหมู่: {summary['categories']}")
print(f"สินค้าเหลือน้อย: {summary['low_stock_items']} รายการ")
```

## 13. สรุป Part 009

ใน Part นี้คุณได้เรียนรู้:

✅ **การสร้าง Dictionary** - 4 วิธี: `{}`, `dict()`, จาก tuples, `fromkeys()`
✅ **CRUD Operations** - อ่านด้วย `[]`/`.get()`, เพิ่ม/แก้ไขด้วย `=`/`.update()`, ลบด้วย `del`/`.pop()`
✅ **Dictionary Methods** - `keys()`, `values()`, `items()`, `get()`, `update()`, `pop()`, `setdefault()`
✅ **Dict Comprehensions** - สร้าง dict แบบ concise พร้อม filter ได้
✅ **defaultdict** - หลีกเลี่ยง KeyError ด้วย default factory
✅ **Counter** - นับความถี่ + arithmetic operations
✅ **OrderedDict** - จัดการลำดับด้วย `move_to_end()`
✅ **Nested Dictionaries** - จัดการ data แบบ hierarchical
✅ **เทคนิคขั้นสูง** - invert dict, grouping, switch/case pattern

---

## ➡️ ถัดไป: Part 010 - Sets

*Part 009/100+ | Python Course - Beginner to World-Class*
