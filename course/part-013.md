# Part 013: Exception Handling
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ try/except/else/finally ได้
- สร้าง Custom Exceptions ได้
- เข้าใจ Exception Hierarchy ได้
- ใช้ raise และ re-raise exceptions ได้
- Log errors ได้อย่างถูกต้อง
- รู้จัก Best Practices ของ Exception Handling

---

## 1. ทำไมต้องจัดการ Exceptions?

```python
# ===== โปรแกรมที่ไม่จัดการ Exception =====
# ถ้าเกิด error โปรแกรมจะหยุดทำงานทันที

def divide_without_handling(a, b):
    return a / b

# เหล่านี้จะ crash โปรแกรม:
# result = divide_without_handling(10, 0)  # ZeroDivisionError
# result = divide_without_handling("10", 2)  # TypeError

# ===== โปรแกรมที่จัดการ Exception =====
def divide_safely(a, b):
    try:
        return a / b
    except ZeroDivisionError:
        print("⚠️  หารด้วยศูนย์ไม่ได้!")
        return None
    except TypeError:
        print("⚠️  ต้องเป็นตัวเลขเท่านั้น!")
        return None

result1 = divide_safely(10, 2)    # 5.0
result2 = divide_safely(10, 0)    # None + warning
result3 = divide_safely("10", 2)  # None + warning

print(f"ผลลัพธ์: {result1}, {result2}, {result3}")
```

## 2. try / except พื้นฐาน

```python
# ===== รูปแบบพื้นฐาน =====
try:
    # โค้ดที่อาจเกิด error
    result = int("not a number")
except ValueError:
    # จัดการถ้าเกิด ValueError
    print("ข้อมูลไม่ถูกต้อง")

# ===== ดึงข้อมูล error =====
try:
    number = int("abc")
except ValueError as e:
    print(f"Error: {e}")              # invalid literal for int()...
    print(f"ชนิด: {type(e).__name__}")  # ValueError

# ===== จัดการหลาย exceptions =====
def process_data(data):
    try:
        # อาจเกิดหลาย errors
        value = int(data)
        result = 100 / value
        items = [1, 2, 3]
        return items[result]
    except ValueError:
        print("ข้อมูลต้องเป็นตัวเลข")
    except ZeroDivisionError:
        print("ห้ามใช้ค่า 0")
    except IndexError:
        print("ดัชนีเกินขอบเขต")

process_data("abc")   # ValueError
process_data(0)       # ZeroDivisionError
process_data(200)     # IndexError

# ===== จับ exceptions หลายตัวพร้อมกัน =====
def parse_config(data):
    try:
        value = int(data)
        return value
    except (ValueError, TypeError) as e:
        print(f"Config error: {e}")
        return None

# ===== จับ Exception ทั่วไป (ระวัง!) =====
try:
    x = 1 / 0
except Exception as e:
    # ⚠️ จับทุก exception - ใช้เฉพาะเมื่อจำเป็น
    print(f"มีบางอย่างผิดพลาด: {type(e).__name__}: {e}")
```

## 3. try / except / else / finally

```python
# ===== else: ทำงานถ้าไม่เกิด exception =====
def read_number(text: str) -> float:
    try:
        value = float(text)
    except ValueError:
        print(f"'{text}' ไม่ใช่ตัวเลข")
        return None
    else:
        # ทำงานเฉพาะถ้า try สำเร็จ
        print(f"แปลง '{text}' เป็น {value} สำเร็จ")
        return value

read_number("3.14")   # แปลงสำเร็จ
read_number("hello")  # ValueError

# ===== finally: ทำงานเสมอ ไม่ว่าจะเกิด error หรือไม่ =====
def open_file(filename: str) -> str:
    f = None
    try:
        f = open(filename, "r", encoding="utf-8")
        content = f.read()
        return content
    except FileNotFoundError:
        print(f"ไม่พบไฟล์: {filename}")
        return ""
    except PermissionError:
        print(f"ไม่มีสิทธิ์อ่านไฟล์: {filename}")
        return ""
    finally:
        # ทำงานเสมอ - ใช้สำหรับ cleanup
        if f:
            f.close()
            print(f"ปิดไฟล์ {filename} แล้ว")

content = open_file("/tmp/demo_write.txt")
content2 = open_file("/tmp/nonexistent.txt")

# ===== ครบทุก clause =====
def connect_database(host: str, port: int):
    connection = None
    try:
        print(f"กำลังเชื่อมต่อ {host}:{port}...")
        if host == "bad_host":
            raise ConnectionError("เชื่อมต่อไม่ได้")
        connection = f"Connection({host}:{port})"
        print("เชื่อมต่อสำเร็จ!")
        return connection
    except ConnectionError as e:
        print(f"❌ Connection Error: {e}")
        return None
    else:
        print("✅ Ready to use database")
    finally:
        print("🔄 Cleanup connection resources")
        if connection:
            print(f"Closing {connection}")

conn = connect_database("localhost", 5432)
conn2 = connect_database("bad_host", 5432)
```

## 4. Exception Hierarchy

```python
"""
Python Exception Hierarchy (บางส่วน):
BaseException
├── SystemExit
├── KeyboardInterrupt
├── GeneratorExit
└── Exception
    ├── ArithmeticError
    │   ├── ZeroDivisionError
    │   ├── OverflowError
    │   └── FloatingPointError
    ├── LookupError
    │   ├── IndexError
    │   └── KeyError
    ├── OSError (IOError)
    │   ├── FileNotFoundError
    │   ├── PermissionError
    │   ├── FileExistsError
    │   └── ConnectionError
    ├── ValueError
    ├── TypeError
    ├── AttributeError
    ├── NameError
    ├── RuntimeError
    ├── StopIteration
    └── ImportError
        └── ModuleNotFoundError
"""

# ===== จับ parent exception ได้ทุก child =====
try:
    items = [1, 2, 3]
    print(items[10])  # IndexError
except LookupError as e:
    print(f"LookupError: {e}")  # จับ IndexError ได้

# ===== ลำดับ except มีความสำคัญ =====
def check_order():
    try:
        d = {}
        d["missing_key"]
    except KeyError:
        print("KeyError - specific")
    except LookupError:
        print("LookupError - general")  # ไม่ถูกเรียกเพราะ KeyError จับก่อน

check_order()

# ⚠️ ถ้าเรียงผิดลำดับ จะได้ general ก่อน
def wrong_order():
    try:
        d = {}
        d["key"]
    except LookupError:    # จับก่อน!
        print("LookupError")
    except KeyError:       # ไม่ถูกเรียกเลย
        print("KeyError")

wrong_order()

# ===== ตรวจสอบ exception type =====
errors = [
    ValueError("bad value"),
    KeyError("missing key"),
    TypeError("wrong type"),
    ZeroDivisionError("div by zero"),
]

for error in errors:
    print(f"{type(error).__name__}: {error}")
    print(f"  isinstance(ValueError): {isinstance(error, ValueError)}")
    print(f"  isinstance(Exception): {isinstance(error, Exception)}")
    print()
```

## 5. raise - สร้าง Exception เอง

```python
# ===== raise Exception =====
def validate_age(age: int) -> None:
    if not isinstance(age, int):
        raise TypeError(f"age ต้องเป็น int ไม่ใช่ {type(age).__name__}")
    if age < 0:
        raise ValueError(f"age ต้องไม่ติดลบ (ได้รับ: {age})")
    if age > 150:
        raise ValueError(f"age ดูไม่สมเหตุสมผล (ได้รับ: {age})")

try:
    validate_age(25)    # ผ่าน
    validate_age(-5)    # ValueError
except ValueError as e:
    print(f"Validation Error: {e}")

try:
    validate_age("25")  # TypeError
except TypeError as e:
    print(f"Type Error: {e}")

# ===== raise from - Exception Chaining =====
def fetch_user_data(user_id: int) -> dict:
    """จำลอง database query"""
    database = {1: {"name": "Alice"}, 2: {"name": "Bob"}}
    try:
        return database[user_id]
    except KeyError as e:
        # สร้าง exception ใหม่ที่มีข้อมูลมากกว่า
        raise ValueError(f"ไม่พบ user_id: {user_id}") from e

try:
    user = fetch_user_data(999)
except ValueError as e:
    print(f"Error: {e}")
    print(f"Caused by: {e.__cause__}")

# ===== Re-raise =====
def process_with_retry(func, max_retries=3):
    """ลอง execute function กี่ครั้ง"""
    for attempt in range(1, max_retries + 1):
        try:
            return func()
        except ValueError as e:
            print(f"Attempt {attempt} failed: {e}")
            if attempt == max_retries:
                raise  # re-raise ต้น exception เดิม
    return None

attempt_count = 0
def unreliable_function():
    global attempt_count
    attempt_count += 1
    if attempt_count < 3:
        raise ValueError(f"ล้มเหลวครั้งที่ {attempt_count}")
    return "สำเร็จ!"

try:
    result = process_with_retry(unreliable_function)
    print(f"ผลลัพธ์: {result}")
except ValueError as e:
    print(f"ล้มเหลวทุก retry: {e}")
```

## 6. Custom Exceptions

```python
# ===== Custom Exception พื้นฐาน =====
class AppError(Exception):
    """Base exception สำหรับแอปพลิเคชัน"""
    pass

class ValidationError(AppError):
    """ข้อมูลไม่ผ่าน validation"""
    pass

class DatabaseError(AppError):
    """ปัญหาเกี่ยวกับ database"""
    pass

class NetworkError(AppError):
    """ปัญหาเครือข่าย"""
    pass

# ===== Custom Exception พร้อมข้อมูลเพิ่มเติม =====
class ValidationError(Exception):
    """Validation error พร้อม field และ message"""
    
    def __init__(self, field: str, message: str, value=None):
        self.field = field
        self.message = message
        self.value = value
        super().__init__(f"[{field}] {message}")
    
    def __str__(self):
        base = f"ValidationError: [{self.field}] {self.message}"
        if self.value is not None:
            base += f" (ได้รับ: {self.value!r})"
        return base

class MultiValidationError(Exception):
    """หลาย validation errors พร้อมกัน"""
    
    def __init__(self, errors: list):
        self.errors = errors
        messages = [str(e) for e in errors]
        super().__init__(f"{len(errors)} validation errors: {'; '.join(messages)}")
    
    def __iter__(self):
        return iter(self.errors)

# ===== ตัวอย่างการใช้ =====
def validate_user(data: dict) -> None:
    """Validate user data และเก็บ errors ทั้งหมดก่อน raise"""
    errors = []
    
    # ตรวจสอบ name
    name = data.get("name", "")
    if not name:
        errors.append(ValidationError("name", "จำเป็นต้องกรอก"))
    elif len(name) < 2:
        errors.append(ValidationError("name", "ต้องมีอย่างน้อย 2 ตัวอักษร", name))
    
    # ตรวจสอบ age
    age = data.get("age")
    if age is None:
        errors.append(ValidationError("age", "จำเป็นต้องกรอก"))
    elif not isinstance(age, int):
        errors.append(ValidationError("age", "ต้องเป็นตัวเลขจำนวนเต็ม", age))
    elif age < 0 or age > 120:
        errors.append(ValidationError("age", "ต้องอยู่ระหว่าง 0-120", age))
    
    # ตรวจสอบ email
    import re
    email = data.get("email", "")
    if not email:
        errors.append(ValidationError("email", "จำเป็นต้องกรอก"))
    elif not re.match(r"[\w.+-]+@[\w-]+\.\w+", email):
        errors.append(ValidationError("email", "รูปแบบไม่ถูกต้อง", email))
    
    if errors:
        raise MultiValidationError(errors)

# ทดสอบ
test_data = [
    {"name": "Alice", "age": 25, "email": "alice@example.com"},  # ผ่าน
    {"name": "B", "age": -5, "email": "bad_email"},               # หลาย errors
    {"name": "", "age": "thirty"},                                  # หลาย errors
]

for data in test_data:
    try:
        validate_user(data)
        print(f"✅ ข้อมูลถูกต้อง: {data.get('name', '(ไม่มีชื่อ)')}")
    except MultiValidationError as e:
        print(f"\n❌ Validation errors ({len(e.errors)} รายการ):")
        for err in e:
            print(f"   - {err}")
```

## 7. Context Manager กับ Exception

```python
# ===== Context Manager สำหรับ resource management =====
class DatabaseConnection:
    """จำลอง database connection"""
    
    def __init__(self, host: str, port: int):
        self.host = host
        self.port = port
        self.connected = False
    
    def __enter__(self):
        print(f"เชื่อมต่อ {self.host}:{self.port}...")
        self.connected = True
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        """
        exc_type: ชนิดของ exception
        exc_val:  ค่า exception
        exc_tb:   traceback
        """
        print(f"ปิดการเชื่อมต่อ...")
        self.connected = False
        
        if exc_type is not None:
            print(f"⚠️  เกิด error: {exc_type.__name__}: {exc_val}")
            # return True  = suppress exception (ไม่ propagate)
            # return False/None = re-raise exception
        
        return False  # ไม่ suppress exception
    
    def query(self, sql: str) -> list:
        if not self.connected:
            raise RuntimeError("ไม่ได้เชื่อมต่อ")
        print(f"  Query: {sql[:50]}...")
        return [{"id": 1, "name": "test"}]

# ===== ใช้งาน =====
print("=== ทำงานปกติ ===")
with DatabaseConnection("localhost", 5432) as db:
    results = db.query("SELECT * FROM users")
    print(f"  ได้ {len(results)} records")
print("หลัง with block\n")

print("=== เกิด Exception ===")
try:
    with DatabaseConnection("localhost", 5432) as db:
        results = db.query("SELECT * FROM users")
        raise ValueError("มีปัญหาใน query")
except ValueError:
    print("จัดการ ValueError แล้ว\n")

# ===== contextlib =====
from contextlib import contextmanager

@contextmanager
def timer(name: str = ""):
    """จับเวลาการทำงาน"""
    import time
    start = time.time()
    try:
        yield  # โค้ดใน with block จะทำงานตรงนี้
    except Exception as e:
        elapsed = time.time() - start
        print(f"⏱️  {name} ล้มเหลวหลัง {elapsed:.3f}s: {e}")
        raise
    else:
        elapsed = time.time() - start
        print(f"⏱️  {name} สำเร็จใน {elapsed:.3f}s")

import time
with timer("Fast operation"):
    time.sleep(0.1)
    result = sum(range(1000000))

try:
    with timer("Failing operation"):
        time.sleep(0.05)
        raise ValueError("เกิดปัญหา!")
except ValueError:
    pass
```

## 8. Logging Errors

```python
import logging
import traceback

# ===== ตั้งค่า Logging =====
logging.basicConfig(
    level=logging.DEBUG,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S"
)

logger = logging.getLogger("myapp")

# ===== Log levels =====
# DEBUG    - ข้อมูลสำหรับ debug
# INFO     - ข้อมูลทั่วไป
# WARNING  - คำเตือน
# ERROR    - error ที่ยังทำงานต่อได้
# CRITICAL - error ร้ายแรง

logger.debug("Debug message")
logger.info("Info message")
logger.warning("Warning message")
logger.error("Error message")
logger.critical("Critical message")

# ===== Log exceptions =====
def divide(a, b):
    try:
        return a / b
    except ZeroDivisionError:
        logger.error("หารด้วยศูนย์: %s / %s", a, b)
        raise
    except TypeError as e:
        logger.error("Type error: %s", e, exc_info=True)  # เพิ่ม traceback
        raise

# ===== Log ไปไฟล์ =====
file_logger = logging.getLogger("file_logger")
file_handler = logging.FileHandler("/tmp/app_errors.log")
file_handler.setLevel(logging.ERROR)
file_handler.setFormatter(logging.Formatter(
    "%(asctime)s [%(levelname)s] %(name)s: %(message)s"
))
file_logger.addHandler(file_handler)

def safe_execute(func, *args, **kwargs):
    """ทำงาน function พร้อม log error"""
    try:
        return func(*args, **kwargs)
    except Exception as e:
        file_logger.error(
            "ล้มเหลวใน %s: %s",
            func.__name__,
            str(e),
            exc_info=True  # เพิ่ม full traceback
        )
        return None

# ===== Structured Error Info =====
def get_error_info(e: Exception) -> dict:
    """สรุปข้อมูล error"""
    return {
        "type": type(e).__name__,
        "message": str(e),
        "module": type(e).__module__,
        "traceback": traceback.format_exc()
    }

try:
    result = 1 / 0
except Exception as e:
    info = get_error_info(e)
    print(f"Error type: {info['type']}")
    print(f"Message: {info['message']}")
```

## 9. Best Practices

```python
# ===== 1. จับ Exception ที่ specific ที่สุด =====

# ❌ ไม่ดี - จับทุกอย่าง
def bad_example():
    try:
        data = {"key": "value"}
        result = data["missing"]
    except:  # bare except - ไม่รู้จะจัดการอะไร
        pass  # ซ่อน error!

# ✅ ดี - จับที่ specific
def good_example():
    try:
        data = {"key": "value"}
        result = data["missing"]
    except KeyError as e:
        print(f"Key ไม่พบ: {e}")
        result = None
    return result

# ===== 2. อย่าทิ้ง exception โดยไม่ทำอะไร =====

# ❌ ไม่ดี
def bad_ignore():
    try:
        int("abc")
    except ValueError:
        pass  # ไม่ทำอะไร!

# ✅ ดี - อย่างน้อย log ไว้
def good_with_log():
    try:
        int("abc")
    except ValueError as e:
        logging.warning("Cannot convert: %s", e)

# ===== 3. ใช้ else สำหรับ "happy path" =====

# ❌ ไม่ดี - โค้ดเยอะใน try
def bad_structure():
    try:
        value = int("42")
        result = value * 2
        formatted = f"ผลลัพธ์คือ {result}"
        return formatted
    except ValueError:
        return "ข้อมูลไม่ถูกต้อง"

# ✅ ดี - แยก conversion กับ processing
def good_structure():
    try:
        value = int("42")
    except ValueError:
        return "ข้อมูลไม่ถูกต้อง"
    else:
        # ทำงานเฉพาะถ้าไม่มี exception
        result = value * 2
        return f"ผลลัพธ์คือ {result}"

# ===== 4. Fail Fast =====
def process_order(order: dict) -> dict:
    """ตรวจสอบก่อน ค่อยทำ"""
    
    # ตรวจสอบก่อน (Guard Clauses)
    if not order:
        raise ValueError("order ต้องไม่ว่าง")
    if "user_id" not in order:
        raise ValueError("ต้องมี user_id")
    if "items" not in order or not order["items"]:
        raise ValueError("ต้องมีสินค้า")
    
    # ถึงตรงนี้ข้อมูลถูกต้องแน่นอน
    total = sum(item["price"] * item["qty"] for item in order["items"])
    return {"order_id": "ORD001", "total": total, "status": "confirmed"}

# ===== 5. Custom Exception Hierarchy =====
class MyAppError(Exception):
    """Base error สำหรับแอป"""
    def __init__(self, message: str, code: int = None):
        super().__init__(message)
        self.code = code

class DatabaseError(MyAppError):
    """Database related errors"""
    pass

class ConnectionError(DatabaseError):
    """Cannot connect to database"""
    pass

class QueryError(DatabaseError):
    """Query execution failed"""
    pass

class AuthError(MyAppError):
    """Authentication errors"""
    pass

class InvalidTokenError(AuthError):
    """Token is invalid"""
    pass

class ExpiredTokenError(AuthError):
    """Token has expired"""
    pass

# จับทั้ง hierarchy
def handle_auth(token: str):
    try:
        # simulate auth check
        if token == "expired":
            raise ExpiredTokenError("Token หมดอายุแล้ว", code=401)
        elif token == "invalid":
            raise InvalidTokenError("Token ไม่ถูกต้อง", code=401)
    except ExpiredTokenError as e:
        return {"error": str(e), "action": "refresh_token", "code": e.code}
    except AuthError as e:
        return {"error": str(e), "action": "login_again", "code": e.code}
    else:
        return {"user": "authenticated"}

print(handle_auth("expired"))
print(handle_auth("invalid"))
print(handle_auth("valid_token"))
```

## 10. ตัวอย่างโปรเจกต์: Robust File Processor

```python
"""
File Processor ที่จัดการ exceptions ครบถ้วน
"""

import json
import csv
import logging
from pathlib import Path
from typing import Union

# ตั้งค่า logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s"
)
logger = logging.getLogger(__name__)


class FileProcessorError(Exception):
    """Base error สำหรับ FileProcessor"""
    pass

class UnsupportedFormatError(FileProcessorError):
    """รูปแบบไฟล์ไม่รองรับ"""
    pass

class FileParseError(FileProcessorError):
    """Parse ไฟล์ไม่สำเร็จ"""
    def __init__(self, filename: str, cause: Exception):
        super().__init__(f"Parse ล้มเหลว: {filename}")
        self.filename = filename
        self.cause = cause


class FileProcessor:
    """อ่านและประมวลผลไฟล์หลายรูปแบบ"""
    
    SUPPORTED_FORMATS = {".json", ".csv", ".txt"}
    
    def __init__(self, base_dir: str = "/tmp"):
        self.base_dir = Path(base_dir)
    
    def read(self, filename: str) -> Union[dict, list, str]:
        """อ่านไฟล์ตาม extension"""
        path = self.base_dir / filename
        
        # ตรวจสอบ extension
        suffix = path.suffix.lower()
        if suffix not in self.SUPPORTED_FORMATS:
            raise UnsupportedFormatError(
                f"ไม่รองรับ {suffix}. รองรับ: {self.SUPPORTED_FORMATS}"
            )
        
        # ตรวจสอบว่าไฟล์มีอยู่
        if not path.exists():
            raise FileNotFoundError(f"ไม่พบไฟล์: {path}")
        
        try:
            if suffix == ".json":
                return self._read_json(path)
            elif suffix == ".csv":
                return self._read_csv(path)
            elif suffix == ".txt":
                return self._read_text(path)
        except (UnsupportedFormatError, FileNotFoundError):
            raise
        except Exception as e:
            raise FileParseError(filename, e) from e
    
    def _read_json(self, path: Path) -> dict:
        with open(path, "r", encoding="utf-8") as f:
            return json.load(f)
    
    def _read_csv(self, path: Path) -> list:
        with open(path, "r", encoding="utf-8-sig") as f:
            return list(csv.DictReader(f))
    
    def _read_text(self, path: Path) -> str:
        return path.read_text(encoding="utf-8")
    
    def write(self, filename: str, data: Union[dict, list, str]) -> bool:
        """เขียนไฟล์พร้อมจัดการ errors"""
        path = self.base_dir / filename
        suffix = path.suffix.lower()
        
        try:
            path.parent.mkdir(parents=True, exist_ok=True)
            
            if suffix == ".json":
                with open(path, "w", encoding="utf-8") as f:
                    json.dump(data, f, ensure_ascii=False, indent=2)
            elif suffix == ".csv" and isinstance(data, list) and data:
                with open(path, "w", newline="", encoding="utf-8-sig") as f:
                    writer = csv.DictWriter(f, fieldnames=data[0].keys())
                    writer.writeheader()
                    writer.writerows(data)
            elif suffix == ".txt":
                path.write_text(str(data), encoding="utf-8")
            else:
                raise UnsupportedFormatError(f"เขียน {suffix} ไม่ได้")
            
            logger.info("เขียนไฟล์สำเร็จ: %s", path)
            return True
            
        except PermissionError:
            logger.error("ไม่มีสิทธิ์เขียนไฟล์: %s", path)
            return False
        except OSError as e:
            logger.error("OS Error เขียนไฟล์: %s", e)
            return False
    
    def batch_read(self, filenames: list) -> dict:
        """อ่านหลายไฟล์พร้อมรายงาน errors"""
        results = {"success": {}, "errors": {}}
        
        for filename in filenames:
            try:
                data = self.read(filename)
                results["success"][filename] = data
                logger.info("อ่านสำเร็จ: %s", filename)
            except FileNotFoundError as e:
                results["errors"][filename] = f"ไม่พบไฟล์: {e}"
            except UnsupportedFormatError as e:
                results["errors"][filename] = f"รูปแบบไม่รองรับ: {e}"
            except FileParseError as e:
                results["errors"][filename] = f"Parse ล้มเหลว: {e.cause}"
        
        return results


# ===== ทดสอบ =====
processor = FileProcessor("/tmp")

# สร้างไฟล์ทดสอบ
import json, csv
with open("/tmp/test.json", "w") as f:
    json.dump({"name": "test", "value": 42}, f)
with open("/tmp/test.txt", "w") as f:
    f.write("Hello World")

print("=== Batch Read Test ===")
results = processor.batch_read([
    "test.json",
    "test.txt",
    "nonexistent.json",
    "invalid.xyz",
])

print(f"\nสำเร็จ ({len(results['success'])} ไฟล์):")
for name, data in results["success"].items():
    print(f"  ✅ {name}: {str(data)[:50]}")

print(f"\nล้มเหลว ({len(results['errors'])} ไฟล์):")
for name, error in results["errors"].items():
    print(f"  ❌ {name}: {error}")
```

## 11. สรุป Part 013

ใน Part นี้คุณได้เรียนรู้:

✅ **try/except** - จับ specific exceptions, ดึงข้อมูล error ด้วย `as`
✅ **try/except/else/finally** - `else` สำหรับ success path, `finally` สำหรับ cleanup
✅ **Exception Hierarchy** - BaseException → Exception → specific errors
✅ **raise** - สร้าง exception เอง, `raise from` สำหรับ exception chaining
✅ **Re-raise** - ใช้ `raise` โดยไม่มี argument เพื่อ propagate ต่อ
✅ **Custom Exceptions** - สร้าง exception classes พร้อมข้อมูลเพิ่มเติม
✅ **Context Manager** - `__enter__`/`__exit__`, `@contextmanager`
✅ **Logging** - `logging.error()`, `exc_info=True`, file handlers
✅ **Best Practices** - Specific exceptions, อย่าซ่อน errors, Fail Fast

---

## ➡️ ถัดไป: Part 014 - Modules and Packages

*Part 013/100+ | Python Course - Beginner to World-Class*
