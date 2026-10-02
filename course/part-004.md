# Part 004: การควบคุมการไหลของโปรแกรม (Control Flow)
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- ใช้ if, elif, else ได้อย่างมีประสิทธิภาพ
- เข้าใจ Nested Conditions
- รู้จัก Match/Case (Python 3.10+)
- สร้าง Guard Clauses เพื่อโค้ดสะอาด
- ประยุกต์ใช้ในโปรแกรมจริง

---

## 1. if Statement พื้นฐาน

```python
# if Statement ง่ายๆ
temperature = 35

if temperature > 30:
    print("อากาศร้อน!")
    print("ควรดื่มน้ำมากๆ")

# ถ้า condition เป็น False ไม่ทำอะไร
if temperature < 0:
    print("น้ำแข็ง!")  # ไม่แสดง
```

### 1.1 if-else

```python
age = 17

if age >= 18:
    print("คุณเป็นผู้ใหญ่")
    print("สามารถโหวตได้")
else:
    print("คุณยังเป็นผู้เยาว์")
    print(f"อีก {18 - age} ปีจะเป็นผู้ใหญ่")
```

### 1.2 if-elif-else

```python
score = 78

if score >= 90:
    grade = "A"
    description = "ดีเยี่ยม"
elif score >= 80:
    grade = "B"
    description = "ดีมาก"
elif score >= 70:
    grade = "C"
    description = "ดี"
elif score >= 60:
    grade = "D"
    description = "พอใช้"
else:
    grade = "F"
    description = "ไม่ผ่าน"

print(f"คะแนน: {score}")
print(f"เกรด: {grade} ({description})")
```

### 1.3 Indentation (การเยื้อง)

```python
# Python ใช้ indentation (4 spaces) แทนวงเล็บ {}
# ❌ ผิด - ไม่มี indentation
if True:
print("Hello")  # IndentationError!

# ❌ ผิด - indentation ไม่สม่ำเสมอ
if True:
    print("Hello")
  print("World")  # IndentationError!

# ✅ ถูกต้อง
if True:
    print("Hello")
    print("World")

# Nested block
x = 10
if x > 0:
    print("Positive")
    if x > 5:
        print("Greater than 5")
        if x > 8:
            print("Greater than 8")  # 3 levels
```

---

## 2. Nested Conditions

```python
# ระบบตรวจสอบสิทธิ์
def check_access(username, password, is_admin, is_active):
    if username and password:
        if is_active:
            if is_admin:
                return "Admin access granted"
            else:
                return "User access granted"
        else:
            return "Account is deactivated"
    else:
        return "Username and password required"

# ทดสอบ
print(check_access("alice", "pass123", True, True))   # Admin access
print(check_access("bob", "pass456", False, True))    # User access
print(check_access("charlie", "pass", True, False))   # Deactivated
print(check_access("", "", True, True))               # No credentials

# ⚠️ Nested ลึกเกินไปอ่านยาก - ดีกว่าใช้ Guard Clauses
```

### 2.1 Guard Clauses (Early Return) - โค้ดสะอาด

```python
# ❌ แบบ nested (อ่านยาก)
def process_order_nested(user, items, payment):
    if user:
        if user.get('is_active'):
            if items:
                if len(items) > 0:
                    if payment:
                        if payment.get('amount') > 0:
                            return "Order processed"
                        else:
                            return "Invalid payment amount"
                    else:
                        return "Payment required"
                else:
                    return "No items"
            else:
                return "No items"
        else:
            return "User account inactive"
    else:
        return "User required"

# ✅ แบบ Guard Clauses (อ่านง่าย)
def process_order(user, items, payment):
    # Guard clauses - ตรวจสอบ conditions ที่ fail ก่อน
    if not user:
        return "User required"
    
    if not user.get('is_active'):
        return "User account inactive"
    
    if not items or len(items) == 0:
        return "No items"
    
    if not payment:
        return "Payment required"
    
    if payment.get('amount', 0) <= 0:
        return "Invalid payment amount"
    
    # Happy path - ส่วนที่ทำงานจริง
    return "Order processed"

# ทดสอบ
user = {'name': 'Alice', 'is_active': True}
items = [{'id': 1, 'name': 'Book', 'price': 250}]
payment = {'method': 'credit_card', 'amount': 250}

print(process_order(user, items, payment))           # Order processed
print(process_order(None, items, payment))           # User required
print(process_order({'is_active': False}, items, payment))  # Inactive
print(process_order(user, [], payment))              # No items
print(process_order(user, items, None))              # Payment required
```

---

## 3. Conditions ขั้นสูง

### 3.1 Multiple Conditions

```python
# BMI Calculator
def get_bmi_status(weight_kg, height_m):
    bmi = weight_kg / (height_m ** 2)
    
    if bmi < 18.5:
        status = "น้ำหนักน้อยเกิน"
    elif 18.5 <= bmi < 25.0:
        status = "น้ำหนักปกติ"
    elif 25.0 <= bmi < 30.0:
        status = "น้ำหนักเกิน"
    else:
        status = "อ้วน"
    
    return bmi, status

bmi, status = get_bmi_status(70, 1.75)
print(f"BMI: {bmi:.1f} - {status}")

# ตรวจสอบหลายเงื่อนไขพร้อมกัน
def get_weather_advice(temp, humidity, rain):
    """ให้คำแนะนำตามสภาพอากาศ"""
    advice = []
    
    if temp > 35:
        advice.append("อากาศร้อนมาก ระวังฮีทสโตรก")
    elif temp < 15:
        advice.append("อากาศหนาว ใส่เสื้อหนา")
    
    if humidity > 80:
        advice.append("ความชื้นสูง รู้สึกร้อนกว่าปกติ")
    
    if rain:
        advice.append("มีฝน พกร่มด้วย")
    
    if not advice:
        advice.append("สภาพอากาศดี เหมาะแก่การออกไปข้างนอก")
    
    return advice

weather_advice = get_weather_advice(38, 85, True)
for a in weather_advice:
    print(f"• {a}")
```

### 3.2 ใช้ in และ not in

```python
# แทนที่การใช้หลาย or
# ❌ แบบยาว
def is_weekend_bad(day):
    if day == "Saturday" or day == "Sunday":
        return True
    return False

# ✅ แบบสั้น
def is_weekend(day):
    return day in ("Saturday", "Sunday")

# ตัวอย่างใช้งาน
def get_day_type(day):
    weekdays = ("Monday", "Tuesday", "Wednesday", "Thursday", "Friday")
    weekends = ("Saturday", "Sunday")
    
    if day in weekdays:
        return "Weekday"
    elif day in weekends:
        return "Weekend"
    else:
        return "Invalid day"

for day in ["Monday", "Saturday", "Holiday"]:
    print(f"{day}: {get_day_type(day)}")

# ตรวจสอบชนิดข้อมูลหลายประเภท
def process_value(value):
    if isinstance(value, (int, float)):
        return f"Number: {value * 2}"
    elif isinstance(value, str):
        return f"String: {value.upper()}"
    elif isinstance(value, (list, tuple)):
        return f"Sequence length: {len(value)}"
    elif isinstance(value, dict):
        return f"Dict keys: {list(value.keys())}"
    else:
        return f"Unknown type: {type(value)}"

print(process_value(42))
print(process_value("hello"))
print(process_value([1, 2, 3]))
print(process_value({"a": 1}))
```

### 3.3 Conditional Expression ใน Context ต่างๆ

```python
# ใน list comprehension
numbers = range(-5, 6)
signs = ["positive" if n > 0 else "negative" if n < 0 else "zero" for n in numbers]
print(signs)

# ใน dictionary comprehension
scores = {"Alice": 85, "Bob": 72, "Charlie": 91, "Diana": 68}
grades = {name: "Pass" if score >= 70 else "Fail" for name, score in scores.items()}
print(grades)

# ใน function return
def max_of_two(a, b):
    return a if a > b else b

# ใน assignment
x = int(input("ตัวเลข: ")) if __name__ == "__main__" else 0

# ซับซ้อน
def classify_number(n):
    return (
        "บวก" if n > 0
        else "ลบ" if n < 0
        else "ศูนย์"
    )
```

---

## 4. Match/Case Statement (Python 3.10+)

Python 3.10 เพิ่ม Structural Pattern Matching ซึ่งคล้าย switch/case ในภาษาอื่น

### 4.1 พื้นฐาน Match/Case

```python
# ต้องใช้ Python 3.10+
def http_status(code):
    match code:
        case 200:
            return "OK"
        case 201:
            return "Created"
        case 204:
            return "No Content"
        case 301 | 302:          # Multiple patterns
            return "Redirect"
        case 400:
            return "Bad Request"
        case 401:
            return "Unauthorized"
        case 403:
            return "Forbidden"
        case 404:
            return "Not Found"
        case 500:
            return "Internal Server Error"
        case _:                  # Wildcard (default)
            return "Unknown Status"

for code in [200, 404, 302, 500, 999]:
    print(f"{code}: {http_status(code)}")
```

### 4.2 Match กับ Patterns ขั้นสูง

```python
# Match กับ string
def process_command(command):
    match command.lower().split():
        case ["quit"]:
            print("Quitting...")
        case ["hello"]:
            print("Hello!")
        case ["go", direction]:
            print(f"Going {direction}")
        case ["pick", "up", item]:
            print(f"Picking up {item}")
        case ["drop", *items]:
            print(f"Dropping: {', '.join(items)}")
        case _:
            print(f"Unknown command: {command}")

process_command("quit")
process_command("hello")
process_command("go north")
process_command("pick up sword")
process_command("drop sword shield bow")
process_command("fly")

# Match กับ Dictionary
def handle_event(event):
    match event:
        case {"type": "click", "x": x, "y": y}:
            print(f"Click at ({x}, {y})")
        case {"type": "keypress", "key": key}:
            print(f"Key pressed: {key}")
        case {"type": "resize", "width": w, "height": h}:
            print(f"Window resized to {w}x{h}")
        case {"type": type_}:
            print(f"Unknown event type: {type_}")
        case _:
            print("Invalid event")

handle_event({"type": "click", "x": 100, "y": 200})
handle_event({"type": "keypress", "key": "Enter"})
handle_event({"type": "resize", "width": 1920, "height": 1080})

# Match กับ class (dataclass)
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float

@dataclass
class Circle:
    center: Point
    radius: float

@dataclass
class Rectangle:
    top_left: Point
    bottom_right: Point

def describe_shape(shape):
    match shape:
        case Point(x=0, y=0):
            return "Origin point"
        case Point(x=x, y=0):
            return f"On X-axis at {x}"
        case Point(x=0, y=y):
            return f"On Y-axis at {y}"
        case Point(x=x, y=y):
            return f"Point at ({x}, {y})"
        case Circle(center=Point(x=x, y=y), radius=r):
            return f"Circle centered at ({x}, {y}) with radius {r}"
        case Rectangle(top_left=Point(x=x1, y=y1), bottom_right=Point(x=x2, y=y2)):
            area = abs(x2-x1) * abs(y2-y1)
            return f"Rectangle from ({x1},{y1}) to ({x2},{y2}), area={area}"
        case _:
            return "Unknown shape"

shapes = [
    Point(0, 0),
    Point(5, 0),
    Point(0, 3),
    Point(3, 4),
    Circle(Point(0, 0), 5),
    Rectangle(Point(0, 0), Point(10, 5)),
]

for shape in shapes:
    print(describe_shape(shape))
```

### 4.3 Match กับ Guards

```python
def classify_number(n):
    match n:
        case 0:
            return "Zero"
        case n if n < 0:
            return f"Negative: {n}"
        case n if n % 2 == 0:
            return f"Positive Even: {n}"
        case _:
            return f"Positive Odd: {n}"

for num in [-5, 0, 4, 7, 100]:
    print(f"{num}: {classify_number(num)}")
```

---

## 5. Condition Patterns ที่พบบ่อย

### 5.1 Validation Pattern

```python
def validate_email(email):
    """ตรวจสอบ email format"""
    if not email:
        return False, "Email ต้องไม่ว่าง"
    
    if "@" not in email:
        return False, "Email ต้องมี @"
    
    parts = email.split("@")
    if len(parts) != 2:
        return False, "Email มี @ ได้แค่ตัวเดียว"
    
    local, domain = parts
    
    if not local:
        return False, "ต้องมีชื่อก่อน @"
    
    if "." not in domain:
        return False, "Domain ต้องมี ."
    
    if domain.endswith(".") or domain.startswith("."):
        return False, "Domain ผิดรูปแบบ"
    
    return True, "Email ถูกต้อง"

emails = [
    "alice@example.com",
    "bob",
    "charlie@",
    "@example.com",
    "dave@@example.com",
    "eve@example.",
    "frank@example.co.th",
]

for email in emails:
    valid, message = validate_email(email)
    status = "✅" if valid else "❌"
    print(f"{status} '{email}': {message}")
```

### 5.2 State Machine Pattern

```python
def process_order_state(state, action):
    """ระบบจัดการ order state"""
    
    valid_transitions = {
        "pending": ["confirm", "cancel"],
        "confirmed": ["ship", "cancel"],
        "shipped": ["deliver", "return"],
        "delivered": ["complete", "return"],
        "returned": ["refund"],
        "completed": [],
        "cancelled": [],
        "refunded": [],
    }
    
    if state not in valid_transitions:
        return None, f"Invalid state: {state}"
    
    if action not in valid_transitions[state]:
        return None, f"Cannot '{action}' from state '{state}'"
    
    # State transitions
    new_state_map = {
        ("pending", "confirm"): "confirmed",
        ("pending", "cancel"): "cancelled",
        ("confirmed", "ship"): "shipped",
        ("confirmed", "cancel"): "cancelled",
        ("shipped", "deliver"): "delivered",
        ("shipped", "return"): "returned",
        ("delivered", "complete"): "completed",
        ("delivered", "return"): "returned",
        ("returned", "refund"): "refunded",
    }
    
    new_state = new_state_map.get((state, action))
    if new_state:
        return new_state, f"Order moved from '{state}' to '{new_state}'"
    
    return None, "Invalid transition"

# จำลองการทำงาน
order_state = "pending"
actions = ["confirm", "ship", "deliver", "complete"]

print(f"เริ่มต้น state: {order_state}\n")
for action in actions:
    new_state, message = process_order_state(order_state, action)
    if new_state:
        order_state = new_state
        print(f"Action: {action} → {message}")
    else:
        print(f"Error: {message}")

print(f"\nสุดท้าย state: {order_state}")
```

### 5.3 Configuration Pattern

```python
def get_database_config(environment):
    """ดึง config ตาม environment"""
    
    base_config = {
        "pool_size": 10,
        "timeout": 30,
        "ssl": False,
    }
    
    if environment == "development":
        return {
            **base_config,
            "host": "localhost",
            "port": 5432,
            "database": "myapp_dev",
            "debug": True,
        }
    elif environment == "testing":
        return {
            **base_config,
            "host": "localhost",
            "port": 5432,
            "database": "myapp_test",
            "debug": True,
            "pool_size": 2,
        }
    elif environment == "production":
        return {
            **base_config,
            "host": "prod-db.example.com",
            "port": 5432,
            "database": "myapp_prod",
            "debug": False,
            "ssl": True,
            "pool_size": 50,
        }
    else:
        raise ValueError(f"Unknown environment: {environment}")

for env in ["development", "testing", "production"]:
    config = get_database_config(env)
    print(f"\n{env.upper()} config:")
    for key, value in config.items():
        print(f"  {key}: {value}")
```

---

## 6. ตัวอย่างโปรแกรมจริง: ระบบให้คะแนนนักเรียน

```python
# grade_system.py
"""
ระบบให้คะแนนนักเรียนพร้อมข้อมูลเพิ่มเติม
"""

def calculate_grade(scores):
    """
    คำนวณเกรดจาก dict ของคะแนน
    scores = {
        'midterm': 30,  # คะแนนเต็ม 30
        'final': 40,    # คะแนนเต็ม 40
        'lab': 20,      # คะแนนเต็ม 20
        'quiz': 10,     # คะแนนเต็ม 10
    }
    """
    total = sum(scores.values())
    max_total = 100  # คะแนนเต็ม 100
    
    if total < 0 or total > max_total:
        return None, "Invalid scores"
    
    # คำนวณเปอร์เซ็นต์
    percentage = (total / max_total) * 100
    
    # กำหนดเกรด
    if percentage >= 80:
        grade = "A"
        gpa = 4.0
    elif percentage >= 75:
        grade = "B+"
        gpa = 3.5
    elif percentage >= 70:
        grade = "B"
        gpa = 3.0
    elif percentage >= 65:
        grade = "C+"
        gpa = 2.5
    elif percentage >= 60:
        grade = "C"
        gpa = 2.0
    elif percentage >= 55:
        grade = "D+"
        gpa = 1.5
    elif percentage >= 50:
        grade = "D"
        gpa = 1.0
    else:
        grade = "F"
        gpa = 0.0
    
    # สถานะ
    if gpa >= 3.5:
        status = "เกียรตินิยม"
    elif gpa >= 2.0:
        status = "ผ่าน"
    elif gpa >= 1.0:
        status = "ผ่านแบบมีเงื่อนไข"
    else:
        status = "ไม่ผ่าน"
    
    return {
        "total": total,
        "percentage": percentage,
        "grade": grade,
        "gpa": gpa,
        "status": status
    }, None

# ตัวอย่างนักเรียน
students = [
    {
        "name": "อลิซ สมาร์ท",
        "scores": {"midterm": 28, "final": 38, "lab": 18, "quiz": 9}
    },
    {
        "name": "บ๊อบ เฉลี่ย",
        "scores": {"midterm": 22, "final": 30, "lab": 15, "quiz": 7}
    },
    {
        "name": "ชาร์ลี อ่อน",
        "scores": {"midterm": 15, "final": 20, "lab": 10, "quiz": 5}
    },
    {
        "name": "ไดอาน่า เก่ง",
        "scores": {"midterm": 30, "final": 40, "lab": 20, "quiz": 10}
    },
]

print("=" * 65)
print(f"{'ชื่อ':<20} {'คะแนน':>6} {'%':>7} {'เกรด':>6} {'GPA':>5} สถานะ")
print("=" * 65)

for student in students:
    result, error = calculate_grade(student['scores'])
    if error:
        print(f"Error for {student['name']}: {error}")
        continue
    
    print(f"{student['name']:<20} "
          f"{result['total']:>6} "
          f"{result['percentage']:>6.1f}% "
          f"{result['grade']:>6} "
          f"{result['gpa']:>5.1f} "
          f"{result['status']}")

print("=" * 65)

# สรุปสถิติ
all_results = [calculate_grade(s['scores'])[0] for s in students]
avg_gpa = sum(r['gpa'] for r in all_results) / len(all_results)
passed = sum(1 for r in all_results if r['gpa'] >= 2.0)

print(f"\nสรุปห้องเรียน:")
print(f"  จำนวนนักเรียน: {len(students)}")
print(f"  ผ่าน/ไม่ผ่าน: {passed}/{len(students) - passed}")
print(f"  GPA เฉลี่ย: {avg_gpa:.2f}")
```

---

## 7. ตัวอย่าง: Simple Chatbot

```python
# simple_chatbot.py
"""
Chatbot ง่ายๆ ที่ใช้ if/elif/else
"""

def chatbot_response(user_input):
    """ตอบโต้กับผู้ใช้"""
    user_input = user_input.lower().strip()
    
    # ทักทาย
    greetings = ["hello", "hi", "สวัสดี", "ดีจ้า", "เฮ้"]
    farewells = ["bye", "goodbye", "ลาก่อน", "แล้วเจอกัน"]
    
    if user_input in greetings:
        return "สวัสดีครับ! มีอะไรให้ช่วยได้บ้าง?"
    
    elif user_input in farewells:
        return "ลาก่อนครับ! โชคดีนะ! 👋"
    
    elif "ชื่อ" in user_input or "name" in user_input:
        return "ผมชื่อ PyBot ครับ เป็น Chatbot ที่เรียนรู้ Python"
    
    elif "อายุ" in user_input or "age" in user_input:
        return "ผมเพิ่งถูกสร้างใหม่ๆ เลยครับ อายุยังน้อยอยู่"
    
    elif "python" in user_input:
        return "Python เยี่ยมมากครับ! ใช้ได้ทั้ง Web, AI, Data Science"
    
    elif "django" in user_input:
        return "Django เป็น web framework ที่ดีมากครับ เหมาะสำหรับ production"
    
    elif "flask" in user_input:
        return "Flask เป็น micro-framework ที่เบาและยืดหยุ่นครับ"
    
    elif "fastapi" in user_input:
        return "FastAPI เป็น framework ที่เร็วมากครับ รองรับ async"
    
    elif any(word in user_input for word in ["ขอบคุณ", "thanks", "thank you"]):
        return "ด้วยความยินดีครับ! 😊"
    
    elif "?" in user_input or "ช่วย" in user_input:
        return "ลองถามเรื่อง Python, Django, Flask, หรือ FastAPI ดูนะครับ"
    
    else:
        return f"ขออภัยครับ ยังไม่เข้าใจ '{user_input}' ลองใหม่นะครับ"

# ทดสอบ Chatbot
test_inputs = [
    "สวัสดี",
    "ชื่ออะไร",
    "python ดีไหม",
    "django คืออะไร",
    "ขอบคุณ",
    "ลาก่อน",
]

print("=== PyBot Demo ===")
for user_input in test_inputs:
    print(f"\nUser: {user_input}")
    response = chatbot_response(user_input)
    print(f"Bot:  {response}")
```

---

## 8. Best Practices สำหรับ Conditions

```python
# 1. ใช้ Guard Clauses แทน Nested if
# ❌
def process(data):
    if data:
        if isinstance(data, dict):
            if 'name' in data:
                return data['name'].upper()

# ✅
def process(data):
    if not data:
        return None
    if not isinstance(data, dict):
        return None
    if 'name' not in data:
        return None
    return data['name'].upper()

# 2. ใช้ dictionary แทนหลาย if/elif
# ❌
def get_day_name_bad(day_num):
    if day_num == 1:
        return "Monday"
    elif day_num == 2:
        return "Tuesday"
    elif day_num == 3:
        return "Wednesday"
    # ... ต่อไปเรื่อยๆ

# ✅
def get_day_name(day_num):
    days = {
        1: "Monday", 2: "Tuesday", 3: "Wednesday",
        4: "Thursday", 5: "Friday", 6: "Saturday", 7: "Sunday"
    }
    return days.get(day_num, "Invalid day")

# 3. หลีกเลี่ยง comparison กับ True/False
# ❌
if is_active == True:
    pass

if is_active == False:
    pass

# ✅
if is_active:
    pass

if not is_active:
    pass

# 4. ใช้ 'is' สำหรับ None
# ❌
if result == None:
    pass

# ✅
if result is None:
    pass

if result is not None:
    pass

# 5. อ่านง่ายด้วยการแยก complex condition
# ❌
if (user.age >= 18 and user.is_active and user.has_subscription and 
    not user.is_banned and (user.country in ALLOWED_COUNTRIES)):
    grant_access()

# ✅
is_adult = user.age >= 18
is_valid_account = user.is_active and not user.is_banned
has_valid_subscription = user.has_subscription
is_allowed_region = user.country in ALLOWED_COUNTRIES

if is_adult and is_valid_account and has_valid_subscription and is_allowed_region:
    grant_access()
```

---

## 9. Exercises (แบบฝึกหัด)

### Exercise 1: ระบบตรวจสอบ Password
```python
def validate_password(password):
    """
    ตรวจสอบ password ว่าผ่านเกณฑ์หรือไม่:
    - ความยาวอย่างน้อย 8 ตัวอักษร
    - มีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว
    - มีตัวพิมพ์เล็กอย่างน้อย 1 ตัว
    - มีตัวเลขอย่างน้อย 1 ตัว
    - มีอักขระพิเศษอย่างน้อย 1 ตัว (!@#$%^&*)
    """
    errors = []
    
    if len(password) < 8:
        errors.append("ต้องมีความยาวอย่างน้อย 8 ตัวอักษร")
    
    if not any(c.isupper() for c in password):
        errors.append("ต้องมีตัวพิมพ์ใหญ่")
    
    if not any(c.islower() for c in password):
        errors.append("ต้องมีตัวพิมพ์เล็ก")
    
    if not any(c.isdigit() for c in password):
        errors.append("ต้องมีตัวเลข")
    
    special_chars = "!@#$%^&*"
    if not any(c in special_chars for c in password):
        errors.append("ต้องมีอักขระพิเศษ (!@#$%^&*)")
    
    if errors:
        return False, errors
    
    # กำหนด strength
    length = len(password)
    if length >= 16:
        strength = "Strong 💪"
    elif length >= 12:
        strength = "Good 👍"
    else:
        strength = "OK 👌"
    
    return True, [f"Password ผ่านเกณฑ์! ระดับ: {strength}"]

# ทดสอบ
passwords = [
    "password",
    "Password1",
    "Password1!",
    "MyStr0ng!Password",
    "Sh0rt!",
]

for pwd in passwords:
    valid, messages = validate_password(pwd)
    status = "✅ PASS" if valid else "❌ FAIL"
    print(f"\n'{pwd}': {status}")
    for msg in messages:
        print(f"  → {msg}")
```

### Exercise 2: เกมทาย
```python
import random

def number_guessing_game():
    """เกมทายตัวเลข"""
    secret = random.randint(1, 100)
    max_attempts = 7
    attempts = 0
    
    print("🎮 เกมทายตัวเลข 1-100")
    print(f"คุณมี {max_attempts} ครั้ง")
    print("-" * 30)
    
    while attempts < max_attempts:
        attempts += 1
        remaining = max_attempts - attempts
        
        try:
            guess = int(input(f"\nครั้งที่ {attempts}: ทายตัวเลข: "))
        except ValueError:
            print("กรุณากรอกตัวเลขจำนวนเต็ม")
            attempts -= 1  # ไม่นับครั้งนี้
            continue
        
        if guess < 1 or guess > 100:
            print("กรุณากรอกตัวเลข 1-100")
            attempts -= 1
            continue
        
        if guess == secret:
            print(f"\n🎉 ถูกต้อง! คำตอบคือ {secret}")
            print(f"คุณทายถูกใน {attempts} ครั้ง!")
            
            if attempts == 1:
                print("🏆 เก่งมาก! ทายถูกครั้งแรก!")
            elif attempts <= 3:
                print("⭐ ดีมาก!")
            elif attempts <= 5:
                print("👍 ดี!")
            else:
                print("😅 โอเค!")
            return
        
        elif guess < secret:
            print(f"น้อยเกินไป! ตัวเลขอยู่ระหว่าง {guess} - 100")
        else:
            print(f"มากเกินไป! ตัวเลขอยู่ระหว่าง 1 - {guess}")
        
        if remaining > 0:
            print(f"เหลืออีก {remaining} ครั้ง")
        
    print(f"\n😔 หมดครั้งแล้ว! คำตอบคือ {secret}")

# number_guessing_game()  # ยกเว้น comment เพื่อเล่นจริง
```

---

## 10. สรุป Part 004

### สิ่งที่เรียนรู้:

✅ **if/elif/else** - การแยกทางโปรแกรม  
✅ **Nested Conditions** - เงื่อนไขซ้อนกัน  
✅ **Guard Clauses** - Early return สำหรับโค้ดสะอาด  
✅ **Match/Case** - Pattern matching (Python 3.10+)  
✅ **Practical Patterns** - Validation, State Machine, Configuration  
✅ **Best Practices** - การเขียน conditions อย่างมีประสิทธิภาพ  

### Quick Reference:

```python
# if/elif/else
if condition1:
    pass
elif condition2:
    pass
else:
    pass

# Guard clause
def func(x):
    if not x:
        return  # exit early
    # happy path

# Match/case (3.10+)
match value:
    case 1:
        pass
    case 2 | 3:
        pass
    case _:
        pass  # default

# Ternary
x = "yes" if condition else "no"

# in check
if value in (1, 2, 3):
    pass
```

---

## ➡️ ถัดไป: Part 005 - ลูป for และ while

ใน Part ถัดไป เราจะเรียนรู้:
- for loop และ range()
- while loop
- break, continue, pass
- else ใน loops
- Nested loops
- List/Dict/Set Comprehensions

---

*Part 004/100+ | Python Course - Beginner to World-Class*
