# Part 013: Exception Handling
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ try/except/else/finally ได้อย่างถูกต้อง
- สร้าง Custom Exceptions ได้
- เข้าใจ Exception Hierarchy ได้
- ใช้ raise และ re-raise ได้
- เขียน Context Managers ได้
- บันทึก Errors ด้วย logging ได้
- รู้ Best Practices ของ Exception Handling

---

## 1. ทำไมต้องมี Exception Handling?

```python
# ถ้าไม่มี exception handling - โปรแกรม crash
# numbers = [1, 2, 3]
# print(numbers[10])     # IndexError: list index out of range
# print(1 / 0)           # ZeroDivisionError: division by zero
# print(int("abc"))      # ValueError: invalid literal

# ด้วย exception handling - จัดการ error ได้
def safe_divide(a, b):
    try:
        result = a / b
        return result
    except ZeroDivisionError:
        return None

print(safe_divide(10, 2))   # 5.0
print(safe_divide(10, 0))   # None

# Exception vs Error
# Exception - เหตุการณ์ที่ทำให้ program หยุดทำงานปกติ
# Error - subtype ของ exception (ปัญหาที่ร้ายแรงกว่า)

# Exception ที่พบบ่อย
try:
    x = int("abc")           # ValueError
except ValueError:
    print("ValueError: ข้อมูลผิดรูปแบบ")

try:
    lst = [1, 2, 3]
    print(lst[10])           # IndexError
except IndexError:
    print("IndexError: index เกินขอบเขต")

try:
    d = {"a": 1}
    print(d["b"])            # KeyError
except KeyError:
    print("KeyError: ไม่พบ key")

try:
    result = "hello" + 42   # TypeError
except TypeError:
    print("TypeError: type ไม่ตรงกัน")

try:
    import non_existent_module  # ModuleNotFoundError
except ModuleNotFoundError:
    print("ModuleNotFoundError: ไม่พบ module")
```

---

## 2. try/except Structure

```python
# พื้นฐาน
try:
    # โค้ดที่อาจเกิด exception
    result = 10 / 0
except ZeroDivisionError:
    # จัดการ ZeroDivisionError
    print("หารด้วยศูนย์ไม่ได้")

# except หลายชนิด
def parse_data(data, index):
    try:
        value = data[index]
        return int(value)
    except IndexError:
        print(f"Index {index} เกินขอบเขต")
    except ValueError:
        print(f"ไม่สามารถแปลง '{value}' เป็น int ได้")
    except TypeError:
        print("data ต้องเป็น list หรือ sequence")

print(parse_data(["1", "2", "abc"], 1))   # 2
print(parse_data(["1", "2", "abc"], 2))   # ValueError
print(parse_data(["1", "2", "abc"], 10))  # IndexError

# จับหลาย exceptions ใน except เดียว
def read_number(s):
    try:
        return int(s)
    except (ValueError, TypeError):
        return 0

print(read_number("42"))    # 42
print(read_number("abc"))   # 0
print(read_number(None))    # 0

# Exception object
try:
    result = 1 / 0
except ZeroDivisionError as e:
    print(f"Error type: {type(e).__name__}")  # ZeroDivisionError
    print(f"Error message: {e}")               # division by zero
    print(f"Args: {e.args}")                   # ('division by zero',)

# except Exception - จับทุกอย่าง (ไม่แนะนำ แต่บางครั้งจำเป็น)
def safe_operation(func, *args):
    try:
        return func(*args)
    except Exception as e:
        print(f"Unexpected error: {type(e).__name__}: {e}")
        return None

print(safe_operation(int, "123"))    # 123
print(safe_operation(int, "abc"))    # error message, None
```

---

## 3. else และ finally

```python
# else - รันเมื่อ try ไม่มี exception
def divide(a, b):
    try:
        result = a / b
    except ZeroDivisionError:
        print("ไม่สามารถหารด้วยศูนย์")
        return None
    else:
        # รันเฉพาะเมื่อ try สำเร็จ (ไม่มี exception)
        print(f"ผลลัพธ์: {result}")
        return result

divide(10, 2)   # ผลลัพธ์: 5.0
divide(10, 0)   # ไม่สามารถหารด้วยศูนย์

# finally - รันเสมอ ไม่ว่าจะมี exception หรือไม่
def read_file(filepath):
    f = None
    try:
        f = open(filepath, "r")
        content = f.read()
        return content
    except FileNotFoundError:
        print(f"ไม่พบไฟล์: {filepath}")
        return None
    except PermissionError:
        print(f"ไม่มีสิทธิ์อ่านไฟล์: {filepath}")
        return None
    finally:
        if f:
            f.close()     # ปิดไฟล์เสมอ
            print("ปิดไฟล์แล้ว")

# with statement ทำสิ่งเดียวกันได้ดีกว่า
def read_file_better(filepath):
    try:
        with open(filepath, "r", encoding="utf-8") as f:
            return f.read()
    except FileNotFoundError:
        return None

# ตัวอย่างครบ try/except/else/finally
def process_number(text):
    print(f"กำลังประมวลผล: {text!r}")
    try:
        n = int(text)
        result = 100 / n
    except ValueError:
        print("  ✗ ValueError: ไม่ใช่ตัวเลข")
    except ZeroDivisionError:
        print("  ✗ ZeroDivisionError: ไม่สามารถหารด้วยศูนย์")
    else:
        print(f"  ✓ ผลลัพธ์: {result}")
    finally:
        print("  → ประมวลผลเสร็จ (ไม่ว่าจะสำเร็จหรือไม่)")
    print()

process_number("5")
process_number("0")
process_number("abc")
```

---

## 4. raise และ Re-raise

```python
# raise - ยิง exception เอง
def set_age(age):
    if not isinstance(age, int):
        raise TypeError(f"age ต้องเป็น int ไม่ใช่ {type(age).__name__}")
    if age < 0 or age > 150:
        raise ValueError(f"age ต้องอยู่ระหว่าง 0-150 ไม่ใช่ {age}")
    return age

try:
    set_age("twenty")
except TypeError as e:
    print(f"TypeError: {e}")

try:
    set_age(-5)
except ValueError as e:
    print(f"ValueError: {e}")

try:
    set_age(25)
    print("OK: age=25")
except (TypeError, ValueError) as e:
    print(f"Error: {e}")

# raise without argument - re-raise exception ปัจจุบัน
def handle_error():
    try:
        x = 1 / 0
    except ZeroDivisionError as e:
        print(f"พบ error: {e}")
        raise   # re-raise exception เดิม

try:
    handle_error()
except ZeroDivisionError:
    print("จัดการ exception ที่ re-raise")

# raise from - Exception Chaining
class DatabaseError(Exception):
    pass

def get_user(user_id):
    try:
        # จำลอง DB query ล้มเหลว
        raise ConnectionError("ไม่สามารถเชื่อมต่อ database")
    except ConnectionError as e:
        raise DatabaseError(f"ดึงข้อมูล user {user_id} ล้มเหลว") from e

try:
    get_user(123)
except DatabaseError as e:
    print(f"Error: {e}")
    print(f"Caused by: {e.__cause__}")  # original exception

# raise from None - ซ่อน original exception
def simplified_error():
    try:
        x = int("abc")
    except ValueError:
        raise RuntimeError("การประมวลผลล้มเหลว") from None  # ซ่อน cause

try:
    simplified_error()
except RuntimeError as e:
    print(f"Error: {e}")
    print(f"Cause: {e.__cause__}")  # None
```

---

## 5. Custom Exceptions

```python
# สร้าง custom exception
class AppError(Exception):
    """Base exception สำหรับ application"""
    pass

class ValidationError(AppError):
    """ข้อมูล input ไม่ถูกต้อง"""
    def __init__(self, field: str, message: str, value=None):
        self.field = field
        self.message = message
        self.value = value
        super().__init__(f"Validation error on '{field}': {message}")

class NotFoundError(AppError):
    """ไม่พบข้อมูลที่ต้องการ"""
    def __init__(self, resource: str, resource_id):
        self.resource = resource
        self.resource_id = resource_id
        super().__init__(f"{resource} with id={resource_id} not found")

class AuthenticationError(AppError):
    """การยืนยันตัวตนล้มเหลว"""
    pass

class AuthorizationError(AppError):
    """ไม่มีสิทธิ์ดำเนินการ"""
    def __init__(self, action: str, resource: str):
        super().__init__(f"Not authorized to {action} {resource}")

class RateLimitError(AppError):
    """เรียกใช้บ่อยเกินไป"""
    def __init__(self, limit: int, window: int):
        self.limit = limit
        self.window = window
        super().__init__(f"Rate limit exceeded: max {limit} requests per {window} seconds")

# ใช้งาน
def create_user(username: str, email: str, age: int):
    if not username or len(username) < 3:
        raise ValidationError("username", "ต้องมีอย่างน้อย 3 ตัวอักษร", username)
    
    if "@" not in email:
        raise ValidationError("email", "รูปแบบ email ไม่ถูกต้อง", email)
    
    if not isinstance(age, int) or age < 0 or age > 120:
        raise ValidationError("age", "อายุต้องเป็นตัวเลข 0-120", age)
    
    return {"username": username, "email": email, "age": age}

def get_user(user_id: int, database: dict):
    if user_id not in database:
        raise NotFoundError("User", user_id)
    return database[user_id]

def delete_user(user_id: int, current_user: dict, database: dict):
    if current_user.get("role") != "admin":
        raise AuthorizationError("delete", f"user {user_id}")
    if user_id not in database:
        raise NotFoundError("User", user_id)
    del database[user_id]

# ทดสอบ
test_cases = [
    ("ab", "alice@example.com", 25),          # username สั้นเกิน
    ("alice", "not-an-email", 25),            # email ผิด
    ("alice", "alice@example.com", 200),      # อายุเกิน
    ("alice", "alice@example.com", 25),       # ถูกต้อง
]

for username, email, age in test_cases:
    try:
        user = create_user(username, email, age)
        print(f"✓ สร้าง user สำเร็จ: {user['username']}")
    except ValidationError as e:
        print(f"✗ {e}")

# Exception hierarchy
db = {1: {"name": "Alice"}, 2: {"name": "Bob"}}
admin = {"name": "Admin", "role": "admin"}
viewer = {"name": "Viewer", "role": "viewer"}

try:
    get_user(99, db)
except NotFoundError as e:
    print(f"\nNotFoundError: {e}")
    print(f"  resource: {e.resource}, id: {e.resource_id}")

try:
    delete_user(1, viewer, db)
except AuthorizationError as e:
    print(f"\nAuthorizationError: {e}")

# จับ base exception
try:
    delete_user(99, admin, db)
except AppError as e:
    print(f"\nAppError: {type(e).__name__}: {e}")
```

---

## 6. Exception Hierarchy

```python
# BaseException
#  ├── SystemExit
#  ├── KeyboardInterrupt
#  ├── GeneratorExit
#  └── Exception
#       ├── ArithmeticError
#       │    ├── ZeroDivisionError
#       │    ├── FloatingPointError
#       │    └── OverflowError
#       ├── AttributeError
#       ├── EOFError
#       ├── ImportError
#       │    └── ModuleNotFoundError
#       ├── LookupError
#       │    ├── IndexError
#       │    └── KeyError
#       ├── MemoryError
#       ├── NameError
#       │    └── UnboundLocalError
#       ├── OSError
#       │    ├── FileExistsError
#       │    ├── FileNotFoundError
#       │    ├── PermissionError
#       │    └── TimeoutError
#       ├── RuntimeError
#       │    └── RecursionError
#       ├── StopIteration
#       ├── TypeError
#       ├── ValueError
#       │    └── UnicodeError
#       └── Warning

# จับด้วย parent class
try:
    lst = [1, 2, 3]
    print(lst[10])
except LookupError:    # จับทั้ง IndexError และ KeyError
    print("LookupError (IndexError หรือ KeyError)")

# ลำดับ except สำคัญ - specific ก่อน general
try:
    d = {"a": 1}
    print(d["b"])
except KeyError as e:
    print(f"KeyError: {e}")       # ← specific
except LookupError:
    print("LookupError")          # ← general (ไม่ถึงถ้า KeyError รับไปแล้ว)

# ⚠️ อย่าจับ BaseException (จะจับ SystemExit, KeyboardInterrupt ด้วย)
try:
    pass
except Exception:   # ✓ ดีกว่า
    pass

# except bare (except:) - ยิ่งไม่ควร
try:
    pass
except:             # ✗ จับทุกอย่างรวม SystemExit!
    pass
```

---

## 7. Context Managers

```python
# Context Manager จัดการ resource อัตโนมัติ
# __enter__ และ __exit__

# สร้าง context manager ด้วย class
class Timer:
    """วัดเวลาการทำงาน"""
    import time
    
    def __init__(self, name=""):
        self.name = name
    
    def __enter__(self):
        import time
        self.start = time.time()
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        import time
        self.elapsed = time.time() - self.start
        if self.name:
            print(f"{self.name}: {self.elapsed:.4f}s")
        return False  # False = propagate exceptions (True = suppress)

with Timer("computation") as t:
    result = sum(range(1000000))

print(f"elapsed: {t.elapsed:.4f}s")

# Context manager ด้วย contextlib.contextmanager
from contextlib import contextmanager

@contextmanager
def managed_file(filepath, mode="r", encoding="utf-8"):
    """Context manager สำหรับไฟล์ที่จัดการ error"""
    f = None
    try:
        f = open(filepath, mode, encoding=encoding)
        yield f
    except FileNotFoundError:
        print(f"ไม่พบไฟล์: {filepath}")
        yield None
    finally:
        if f:
            f.close()

with managed_file("existing.txt", "w") as f:
    if f:
        f.write("Hello!")

with managed_file("nonexistent.txt") as f:
    if f:
        content = f.read()

# Database transaction context manager
@contextmanager
def transaction(connection):
    """จัดการ database transaction"""
    try:
        yield connection
        connection.commit()  # commit ถ้าสำเร็จ
        print("Transaction committed")
    except Exception as e:
        connection.rollback()  # rollback ถ้าเกิด error
        print(f"Transaction rolled back: {e}")
        raise

# Suppress exceptions ด้วย contextlib.suppress
from contextlib import suppress

with suppress(FileNotFoundError):
    import os
    os.remove("nonexistent_file.txt")  # ไม่ error

# Nested context managers
from contextlib import ExitStack

# เปิดหลายไฟล์พร้อมกันแบบ dynamic
def process_multiple_files(filenames):
    with ExitStack() as stack:
        files = [
            stack.enter_context(
                open(f, "w", encoding="utf-8")
            )
            for f in filenames
        ]
        for i, f in enumerate(files):
            f.write(f"File {i+1}\n")
    print("ปิดไฟล์ทั้งหมดแล้ว")

process_multiple_files(["file1.txt", "file2.txt", "file3.txt"])

# Cleanup
import os
for f in ["existing.txt", "file1.txt", "file2.txt", "file3.txt"]:
    if os.path.exists(f):
        os.remove(f)
```

---

## 8. Logging Errors

```python
import logging
from pathlib import Path

# ตั้งค่า logging
logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s [%(levelname)s] %(name)s: %(message)s',
    datefmt='%Y-%m-%d %H:%M:%S',
)

# Logger levels: DEBUG < INFO < WARNING < ERROR < CRITICAL
logger = logging.getLogger(__name__)

logger.debug("Debug message - รายละเอียดสำหรับ developer")
logger.info("Info message - ข้อมูลทั่วไป")
logger.warning("Warning - สิ่งที่ควรระวัง")
logger.error("Error - เกิดข้อผิดพลาด")
logger.critical("Critical - ปัญหาร้ายแรง")

# logging กับ exception
def risky_operation(x):
    try:
        return 100 / x
    except ZeroDivisionError:
        logger.error("หารด้วยศูนย์", exc_info=True)  # exc_info=True เพิ่ม traceback
        return None

risky_operation(0)

# ตั้งค่า multiple handlers
def setup_logger(name: str, log_file: str = None) -> logging.Logger:
    """ตั้งค่า logger ที่มีทั้ง console และ file"""
    log = logging.getLogger(name)
    log.setLevel(logging.DEBUG)
    
    formatter = logging.Formatter(
        '%(asctime)s [%(levelname)s] %(name)s - %(message)s'
    )
    
    # Console handler
    console = logging.StreamHandler()
    console.setLevel(logging.INFO)
    console.setFormatter(formatter)
    log.addHandler(console)
    
    # File handler
    if log_file:
        file_handler = logging.FileHandler(log_file, encoding="utf-8")
        file_handler.setLevel(logging.DEBUG)
        file_handler.setFormatter(formatter)
        log.addHandler(file_handler)
    
    return log

app_logger = setup_logger("myapp", "app.log")
app_logger.info("Application started")
app_logger.debug("Debug details")
app_logger.warning("Something unusual happened")

# cleanup
import os
if os.path.exists("app.log"):
    os.remove("app.log")
```

---

## 9. Best Practices

```python
# 1. จับ exceptions ที่ specific (ไม่ bare except)
# ✗ ผิด
try:
    x = int("abc")
except:
    pass

# ✓ ถูก
try:
    x = int("abc")
except ValueError:
    x = 0

# 2. ไม่ใช้ exception สำหรับ flow control ปกติ
# ✗ ผิด - ใช้ exception แทน if
def get_value_bad(d, key):
    try:
        return d[key]
    except KeyError:
        return None

# ✓ ถูก - ใช้ get()
def get_value_good(d, key):
    return d.get(key)

# 3. Log หรือ handle exceptions อย่างเหมาะสม
import logging
log = logging.getLogger(__name__)

# ✗ ผิด - กลืน exception
try:
    x = 1 / 0
except ZeroDivisionError:
    pass  # ซ่อน error ทั้งหมด

# ✓ ถูก - log แล้วคืนค่า default
try:
    x = 1 / 0
except ZeroDivisionError:
    log.warning("หารด้วยศูนย์ คืนค่า default")
    x = 0

# 4. Custom exceptions ต้องมี meaningful messages
class InsufficientFundsError(Exception):
    def __init__(self, balance: float, amount: float):
        self.balance = balance
        self.amount = amount
        super().__init__(
            f"ยอดเงินไม่พอ: มี {balance:.2f} บาท แต่ต้องการ {amount:.2f} บาท"
        )

# 5. Clean up resources ใน finally หรือ context manager
def process_file(filepath):
    # ✓ ใช้ with statement
    try:
        with open(filepath, "r") as f:
            return f.read()
    except FileNotFoundError:
        return None

# 6. Raise exceptions at appropriate level
class UserService:
    def get_user(self, user_id):
        # business logic
        if user_id <= 0:
            raise ValueError(f"user_id ต้องเป็นบวก ไม่ใช่ {user_id}")
        # ...

# 7. ใช้ assert สำหรับ internal invariants เท่านั้น
def calculate_average(numbers):
    assert len(numbers) > 0, "ต้องมี numbers อย่างน้อย 1 ตัว"
    return sum(numbers) / len(numbers)
# ⚠️ assert ถูกปิดได้ด้วย python -O (optimize mode)
# ใช้ assert สำหรับ debug เท่านั้น ไม่ใช่ validation จาก user input

# 8. Exception groups (Python 3.11+)
# try:
#     ...
# except* ValueError as eg:
#     for e in eg.exceptions:
#         print(e)
```

---

## 10. ตัวอย่างโปรแกรมจริง: Robust API Client

```python
"""
API Client ที่จัดการ exception อย่างถูกต้อง
"""
import json
import time
import logging
from typing import Any, Dict, Optional
from contextlib import contextmanager

log = logging.getLogger(__name__)

# Custom exceptions
class APIError(Exception):
    """Base exception สำหรับ API errors"""
    def __init__(self, message: str, status_code: int = None, response: dict = None):
        super().__init__(message)
        self.status_code = status_code
        self.response = response or {}

class NetworkError(APIError):
    """Network connectivity issues"""
    pass

class AuthError(APIError):
    """Authentication/Authorization failed"""
    pass

class NotFoundError(APIError):
    """Resource not found"""
    pass

class RateLimitError(APIError):
    """Too many requests"""
    def __init__(self, retry_after: int = 60):
        self.retry_after = retry_after
        super().__init__(f"Rate limit exceeded. Retry after {retry_after}s", 429)

class ServerError(APIError):
    """Server-side errors"""
    pass

class APIClient:
    def __init__(self, base_url: str, api_key: str, max_retries: int = 3):
        self.base_url = base_url
        self.api_key = api_key
        self.max_retries = max_retries
        self._session_active = False
    
    def __enter__(self):
        self._session_active = True
        log.info(f"เริ่ม session: {self.base_url}")
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self._session_active = False
        if exc_type:
            log.error(f"Session ปิดด้วย error: {exc_val}")
        else:
            log.info("Session ปิดปกติ")
        return False
    
    def _handle_response(self, status_code: int, response_data: dict) -> dict:
        """จัดการ response status codes"""
        if status_code == 200:
            return response_data
        elif status_code == 401:
            raise AuthError("Authentication ล้มเหลว", status_code, response_data)
        elif status_code == 403:
            raise AuthError("ไม่มีสิทธิ์", status_code, response_data)
        elif status_code == 404:
            raise NotFoundError("ไม่พบ resource", status_code, response_data)
        elif status_code == 429:
            retry_after = int(response_data.get("retry_after", 60))
            raise RateLimitError(retry_after)
        elif 500 <= status_code < 600:
            raise ServerError(f"Server error: {status_code}", status_code, response_data)
        else:
            raise APIError(f"Unexpected status: {status_code}", status_code, response_data)
    
    def request(self, method: str, endpoint: str, **kwargs) -> dict:
        """ส่ง API request พร้อม retry logic"""
        url = f"{self.base_url}{endpoint}"
        
        for attempt in range(1, self.max_retries + 1):
            try:
                log.debug(f"[{attempt}/{self.max_retries}] {method} {url}")
                
                # จำลอง HTTP request
                status_code, data = self._simulate_request(method, endpoint)
                return self._handle_response(status_code, data)
                
            except RateLimitError as e:
                log.warning(f"Rate limited. รอ {e.retry_after}s...")
                if attempt < self.max_retries:
                    time.sleep(0.1)  # จำลอง (ใช้ retry_after จริง)
                    continue
                raise
            
            except NetworkError as e:
                log.error(f"Network error (attempt {attempt}): {e}")
                if attempt < self.max_retries:
                    time.sleep(0.1 * attempt)  # exponential backoff
                    continue
                raise
            
            except (AuthError, NotFoundError):
                raise  # ไม่ retry สำหรับ errors เหล่านี้
            
            except ServerError as e:
                log.error(f"Server error (attempt {attempt}): {e}")
                if attempt < self.max_retries:
                    time.sleep(0.5 * attempt)
                    continue
                raise
        
        raise APIError("เกินจำนวน retries สูงสุด")
    
    def _simulate_request(self, method: str, endpoint: str):
        """จำลอง HTTP requests"""
        responses = {
            "/users": (200, {"users": [{"id": 1, "name": "Alice"}, {"id": 2, "name": "Bob"}]}),
            "/users/1": (200, {"id": 1, "name": "Alice", "email": "alice@example.com"}),
            "/users/999": (404, {"error": "User not found"}),
            "/admin": (403, {"error": "Forbidden"}),
            "/error": (500, {"error": "Internal server error"}),
        }
        return responses.get(endpoint, (404, {"error": "Not found"}))
    
    def get_users(self):
        return self.request("GET", "/users")
    
    def get_user(self, user_id: int):
        return self.request("GET", f"/users/{user_id}")

# ทดสอบ
print("=== API Client Test ===")

with APIClient("https://api.example.com", "secret-key") as client:
    # ดึง users ทั้งหมด
    try:
        users = client.get_users()
        print(f"Users: {users}")
    except APIError as e:
        print(f"Error: {e}")
    
    # ดึง user เฉพาะ
    try:
        user = client.get_user(1)
        print(f"User 1: {user}")
    except NotFoundError as e:
        print(f"Not Found: {e}")
    except APIError as e:
        print(f"API Error: {e}")
    
    # ดึง user ที่ไม่มี
    try:
        user = client.get_user(999)
    except NotFoundError as e:
        print(f"Expected: {e} (status={e.status_code})")
    except APIError as e:
        print(f"Unexpected: {e}")
```

---

## 11. Exercises

### Exercise 1: Input Validator

```python
"""
สร้าง Input Validator ที่:
1. validate หลาย fields พร้อมกัน
2. รวบรวม errors ทั้งหมด (ไม่หยุดที่ error แรก)
3. คืน ValidationResult ที่มี errors ทั้งหมด
"""
from typing import Any, Dict, List, Optional
import re

class ValidationError(Exception):
    def __init__(self, errors: Dict[str, List[str]]):
        self.errors = errors
        super().__init__(f"Validation failed: {len(errors)} field(s) invalid")

class Validator:
    def __init__(self):
        self.errors: Dict[str, List[str]] = {}
    
    def _add_error(self, field: str, message: str):
        self.errors.setdefault(field, []).append(message)
    
    def required(self, field: str, value: Any) -> "Validator":
        if value is None or (isinstance(value, str) and not value.strip()):
            self._add_error(field, "ห้ามเว้นว่าง")
        return self
    
    def min_length(self, field: str, value: str, min_len: int) -> "Validator":
        if isinstance(value, str) and len(value) < min_len:
            self._add_error(field, f"ต้องมีอย่างน้อย {min_len} ตัวอักษร")
        return self
    
    def max_length(self, field: str, value: str, max_len: int) -> "Validator":
        if isinstance(value, str) and len(value) > max_len:
            self._add_error(field, f"ต้องมีไม่เกิน {max_len} ตัวอักษร")
        return self
    
    def email(self, field: str, value: str) -> "Validator":
        pattern = r'^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'
        if isinstance(value, str) and not re.match(pattern, value):
            self._add_error(field, "รูปแบบ email ไม่ถูกต้อง")
        return self
    
    def range(self, field: str, value: Any, min_val, max_val) -> "Validator":
        try:
            n = float(value)
            if n < min_val or n > max_val:
                self._add_error(field, f"ต้องอยู่ระหว่าง {min_val} และ {max_val}")
        except (TypeError, ValueError):
            self._add_error(field, "ต้องเป็นตัวเลข")
        return self
    
    def validate(self) -> None:
        if self.errors:
            raise ValidationError(self.errors)

# ทดสอบ
def register_user(data: dict):
    v = Validator()
    (v.required("username", data.get("username"))
      .min_length("username", data.get("username", ""), 3)
      .max_length("username", data.get("username", ""), 50)
      .required("email", data.get("email"))
      .email("email", data.get("email", ""))
      .required("age", data.get("age"))
      .range("age", data.get("age"), 0, 150))
    
    try:
        v.validate()
        print(f"✓ ลงทะเบียนสำเร็จ: {data['username']}")
        return True
    except ValidationError as e:
        print(f"✗ Validation ล้มเหลว:")
        for field, errors in e.errors.items():
            for error in errors:
                print(f"  - {field}: {error}")
        return False

# ทดสอบ
test_cases = [
    {"username": "ab", "email": "bad-email", "age": 200},  # หลาย errors
    {"username": "", "email": "", "age": None},              # ทุก field ว่าง
    {"username": "alice", "email": "alice@example.com", "age": 25},  # ถูกต้อง
]

for data in test_cases:
    print(f"\nInput: {data}")
    register_user(data)
```

### Exercise 2: Retry Decorator

```python
"""
สร้าง Retry Decorator ที่:
1. retry function อัตโนมัติเมื่อเกิด exception
2. กำหนด max_retries, delay, backoff
3. กำหนดประเภท exception ที่จะ retry
"""
import time
import random
import functools
from typing import Tuple, Type

def retry(
    max_retries: int = 3,
    delay: float = 1.0,
    backoff: float = 2.0,
    exceptions: Tuple[Type[Exception], ...] = (Exception,),
    on_retry=None
):
    """Decorator สำหรับ retry function อัตโนมัติ"""
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            
            for attempt in range(1, max_retries + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    last_exception = e
                    
                    if attempt == max_retries:
                        break
                    
                    wait_time = delay * (backoff ** (attempt - 1))
                    
                    if on_retry:
                        on_retry(attempt, max_retries, e, wait_time)
                    
                    time.sleep(wait_time * 0.01)  # จำลอง (ย่อเวลา)
            
            raise last_exception
        
        return wrapper
    return decorator

# จำลอง flaky service
call_count = 0

@retry(
    max_retries=3,
    delay=0.5,
    backoff=2.0,
    exceptions=(ConnectionError, TimeoutError),
    on_retry=lambda attempt, max_r, e, wait: print(f"  Retry {attempt}/{max_r}: {e} (รอ {wait:.2f}s)")
)
def unstable_api_call(endpoint: str):
    """API ที่ไม่เสถียร - ล้มเหลว 2 ครั้งแรก"""
    global call_count
    call_count += 1
    
    if call_count <= 2:
        raise ConnectionError(f"Connection refused (attempt {call_count})")
    
    call_count = 0  # reset
    return {"status": "success", "data": f"Response from {endpoint}"}

print("=== Retry Decorator Test ===")
try:
    result = unstable_api_call("/api/users")
    print(f"Success: {result}")
except (ConnectionError, TimeoutError) as e:
    print(f"Failed after all retries: {e}")

# ทดสอบที่ล้มเหลวตลอด
always_fail_count = 0

@retry(max_retries=3, delay=0.1, exceptions=(ValueError,))
def always_fail():
    global always_fail_count
    always_fail_count += 1
    raise ValueError(f"Always fails (attempt {always_fail_count})")

try:
    always_fail()
except ValueError as e:
    print(f"\nExpected failure: {e}")
    print(f"Called {always_fail_count} times")
```

---

## 12. สรุป Part 013

### สิ่งที่เรียนรู้:

✅ **try/except** - จัดการ exceptions  
✅ **except หลายชนิด** - จับหลาย exception types  
✅ **else** - รันเมื่อ try สำเร็จ  
✅ **finally** - รันเสมอ (cleanup)  
✅ **raise** - ยิง exception  
✅ **raise from** - Exception chaining  
✅ **Custom Exceptions** - สร้าง hierarchy  
✅ **Context Manager** - `with`, `__enter__`, `__exit__`  
✅ **@contextmanager** - decorator จาก contextlib  
✅ **logging** - บันทึก error อย่างถูกต้อง  
✅ **Best Practices** - specific except, log, cleanup  

### Quick Reference:

```python
# try/except/else/finally
try:
    risky_code()
except SpecificError as e:
    handle_error(e)
except (Error1, Error2):
    handle_multiple()
else:
    success_code()
finally:
    cleanup()

# Custom exception
class MyError(Exception):
    def __init__(self, msg, code=None):
        super().__init__(msg)
        self.code = code

# raise
raise MyError("Something went wrong", code=42)
raise  # re-raise
raise NewError("...") from original_error

# Context manager
class Resource:
    def __enter__(self): return self
    def __exit__(self, exc_type, exc_val, tb): return False

with Resource() as r: ...

# @contextmanager
from contextlib import contextmanager
@contextmanager
def managed():
    try: yield resource
    finally: cleanup()
```

---

## ➡️ ถัดไป: Part 014 - Modules และ Packages

ใน Part ถัดไป เราจะเรียนรู้:
- import statements ทุกรูปแบบ
- สร้าง packages ของตัวเอง
- sys.path และ __name__
- Standard library modules ที่สำคัญ

---

*Part 013/100+ | Python Course - Beginner to World-Class*
