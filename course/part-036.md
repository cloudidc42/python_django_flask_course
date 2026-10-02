# Part 036: Logging
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ logging module และ log levels
- ใช้ basicConfig สำหรับ simple logging
- FileHandler และ StreamHandler
- Formatters สำหรับ log format
- getLogger สร้าง named loggers
- Rotating file handlers
- Logging ใน library/application
- ตัวอย่างจริง: logging ใน web app

---

## 1. ทำไมต้องใช้ Logging?

```python
# ❌ แบบผิด: ใช้ print() ทุกที่
def process_payment(amount, user_id):
    print(f"Processing payment {amount} for user {user_id}")
    # ... logic
    print("Payment done")

# ปัญหาของ print():
# - ไม่มี timestamp
# - ไม่มี severity level (debug vs error)
# - ไม่สามารถ filter ได้
# - ไม่สามารถส่งไป file/service ได้
# - ยากต่อการ disable ใน production

# ✅ แบบถูก: ใช้ logging
import logging

def process_payment_good(amount: float, user_id: int) -> bool:
    logger = logging.getLogger(__name__)
    
    logger.info(f"Processing payment {amount:.2f} THB for user {user_id}")
    
    try:
        # ... logic
        logger.debug(f"Payment validation passed for amount={amount}")
        logger.info(f"Payment successful for user {user_id}")
        return True
    except Exception as e:
        logger.error(f"Payment failed for user {user_id}: {e}", exc_info=True)
        return False

# ข้อดีของ logging:
# ✅ Timestamps อัตโนมัติ
# ✅ Log levels สำหรับ filtering
# ✅ ส่งไปหลาย destinations (file, console, server)
# ✅ ปิด/เปิดได้โดยไม่แก้โค้ด
# ✅ รู้ว่า log มาจาก module/function ไหน
```

---

## 2. Log Levels

```python
import logging

# 5 Log Levels (น้อย → มาก):
# DEBUG    (10) - ข้อมูล detailed สำหรับ debugging
# INFO     (20) - ข้อมูลทั่วไป, ยืนยัน flow ปกติ
# WARNING  (30) - บางอย่างผิดปกติแต่ยังทำงานได้
# ERROR    (40) - error ที่ทำให้บาง function ไม่ทำงาน
# CRITICAL (50) - error ร้ายแรงที่อาจทำให้โปรแกรมหยุด

# แสดง level value
for level_name in ["DEBUG", "INFO", "WARNING", "ERROR", "CRITICAL"]:
    level = logging.getLevelName(level_name)
    print(f"{level_name:10} = {level}")
# DEBUG      = 10
# INFO       = 20
# WARNING    = 30
# ERROR      = 40
# CRITICAL   = 50

# basicConfig - ตั้งค่า root logger
logging.basicConfig(
    level=logging.DEBUG,
    format="%(asctime)s [%(levelname)s] %(message)s"
)

# ทดสอบทุก levels
logging.debug("Debug: เห็นเฉพาะตอน development")
logging.info("Info: Flow ปกติของโปรแกรม")
logging.warning("Warning: บางอย่างอาจเป็นปัญหา")
logging.error("Error: บางอย่างผิดพลาด")
logging.critical("Critical: ระบบอาจล่ม!")

# ตัวอย่างการใช้แต่ละ level
import os, sys

def startup_checks() -> bool:
    logging.info("Starting application...")
    
    # DEBUG: ข้อมูล detail
    logging.debug(f"Python version: {sys.version}")
    logging.debug(f"Working directory: {os.getcwd()}")
    
    # WARNING: config ที่ไม่ดี
    if not os.environ.get("SECRET_KEY"):
        logging.warning("SECRET_KEY not set! Using default (not secure for production)")
    
    db_url = os.environ.get("DATABASE_URL", "sqlite:///dev.db")
    logging.info(f"Database: {db_url}")
    
    return True

startup_checks()
```

---

## 3. basicConfig และ Format

```python
import logging

# basicConfig options
logging.basicConfig(
    level=logging.DEBUG,
    
    # Format string
    format="%(asctime)s [%(levelname)-8s] %(name)s:%(lineno)d - %(message)s",
    
    # Date format
    datefmt="%Y-%m-%d %H:%M:%S",
    
    # Handlers
    handlers=[
        logging.StreamHandler(),  # output ไป console
    ]
)

# Format placeholders:
# %(asctime)s    - timestamp
# %(levelname)s  - log level name (DEBUG, INFO, ...)
# %(name)s       - logger name
# %(message)s    - log message
# %(filename)s   - filename
# %(funcName)s   - function name
# %(lineno)d     - line number
# %(thread)d     - thread ID
# %(process)d    - process ID
# %(module)s     - module name

logger = logging.getLogger("myapp")
logger.info("Application started")
logger.debug("Config loaded")
logger.warning("Low memory warning")

# ตัวอย่าง formats
formats = {
    "Simple": "%(levelname)s: %(message)s",
    "With time": "%(asctime)s - %(levelname)s - %(message)s",
    "Detailed": "%(asctime)s [%(levelname)-8s] %(name)s:%(lineno)d - %(message)s",
    "JSON-like": '{"time":"%(asctime)s","level":"%(levelname)s","msg":"%(message)s"}',
}

for name, fmt in formats.items():
    print(f"\n{name}:")
    formatter = logging.Formatter(fmt, datefmt="%H:%M:%S")
    # Test formatter
    record = logging.LogRecord(
        name="test", level=logging.INFO,
        pathname="app.py", lineno=42,
        msg="Hello World", args=(), exc_info=None
    )
    print(f"  {formatter.format(record)}")
```

---

## 4. Handlers

```python
import logging
import sys
import os

# Handler = ปลายทางที่ log ถูกส่งไป
# StreamHandler  - console (stdout/stderr)
# FileHandler    - บันทึกลงไฟล์
# RotatingFileHandler - บันทึกลงไฟล์ที่ rotate
# TimedRotatingFileHandler - rotate ตามเวลา
# NullHandler    - ทิ้ง logs ทั้งหมด (สำหรับ library)

def setup_logging(log_file: str = "app.log") -> logging.Logger:
    """ตั้งค่า logger ที่สมบูรณ์"""
    
    logger = logging.getLogger("myapp")
    logger.setLevel(logging.DEBUG)
    
    # ป้องกัน handlers ซ้ำ
    if logger.handlers:
        return logger
    
    # Formatter
    formatter = logging.Formatter(
        fmt="%(asctime)s [%(levelname)-8s] %(name)s - %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S"
    )
    
    # StreamHandler - console (INFO ขึ้นไป)
    console_handler = logging.StreamHandler(sys.stdout)
    console_handler.setLevel(logging.INFO)
    console_handler.setFormatter(formatter)
    
    # FileHandler - บันทึกทุก level ลงไฟล์
    file_handler = logging.FileHandler(log_file, encoding="utf-8")
    file_handler.setLevel(logging.DEBUG)
    file_handler.setFormatter(formatter)
    
    # Error file - เฉพาะ ERROR ขึ้นไป
    error_handler = logging.FileHandler("errors.log", encoding="utf-8")
    error_handler.setLevel(logging.ERROR)
    error_handler.setFormatter(logging.Formatter(
        "%(asctime)s [%(levelname)s] %(name)s:%(lineno)d\n"
        "  Message: %(message)s\n"
        "  %(exc_text)s\n"
        "---"
    ))
    
    logger.addHandler(console_handler)
    logger.addHandler(file_handler)
    logger.addHandler(error_handler)
    
    return logger

logger = setup_logging("myapp.log")

logger.debug("Debug info (เห็นเฉพาะในไฟล์)")
logger.info("Server started on port 8080")
logger.warning("Deprecated API called")

try:
    result = 1 / 0
except ZeroDivisionError:
    logger.error("Math error occurred", exc_info=True)

# exc_info=True → บันทึก stack trace ด้วย

# ทำความสะอาดไฟล์ test
for f in ["myapp.log", "errors.log"]:
    if os.path.exists(f):
        os.remove(f)
```

---

## 5. Named Loggers และ Hierarchy

```python
import logging

# Logger Hierarchy:
# root logger
#   ├── myapp
#   │   ├── myapp.database
#   │   ├── myapp.api
#   │   └── myapp.auth

# getLogger(__name__) สร้าง logger ที่มีชื่อตาม module
# เมื่อ module ชื่อ myapp.api → logger ชื่อ myapp.api

# Root logger
root = logging.getLogger()
print(f"Root logger: {root.name}")

# Application loggers
app_logger = logging.getLogger("myapp")
db_logger = logging.getLogger("myapp.database")
api_logger = logging.getLogger("myapp.api")
auth_logger = logging.getLogger("myapp.auth")

# Logger propagation: child → parent → root
# db_logger → myapp_logger → root_logger

# ตั้งค่า parent
app_logger.setLevel(logging.DEBUG)
handler = logging.StreamHandler()
handler.setFormatter(logging.Formatter("%(name)s [%(levelname)s]: %(message)s"))
app_logger.addHandler(handler)

# Child loggers สืบทอด handler จาก parent
db_logger.info("Connected to database")    # ใช้ handler ของ myapp
api_logger.warning("Rate limit exceeded")  # ใช้ handler ของ myapp
auth_logger.debug("Token validated")       # ใช้ handler ของ myapp

# ตั้งค่า level สำหรับ specific logger
db_logger.setLevel(logging.WARNING)  # ซ่อน DEBUG/INFO ของ database logger
db_logger.info("This won't show")    # ถูก filter
db_logger.warning("DB warning")      # แสดง

# propagate=False: หยุดส่ง log ไป parent
noisy_lib = logging.getLogger("noisy_library")
noisy_lib.propagate = False  # ไม่ propagate ไป root
noisy_lib.addHandler(logging.NullHandler())  # ทิ้งทั้งหมด

# ตัวอย่าง module-level loggers
# --- file: myapp/database.py ---
# import logging
# logger = logging.getLogger(__name__)  # "myapp.database"
# def connect(): logger.info("Connecting...")

# --- file: myapp/api.py ---
# import logging
# logger = logging.getLogger(__name__)  # "myapp.api"
# def handle_request(): logger.debug("Request received")
```

---

## 6. RotatingFileHandler

```python
import logging
from logging.handlers import RotatingFileHandler, TimedRotatingFileHandler
import os

# RotatingFileHandler: rotate เมื่อไฟล์ใหญ่เกินขนาดที่กำหนด
def setup_rotating_logger(name: str) -> logging.Logger:
    logger = logging.getLogger(name)
    logger.setLevel(logging.DEBUG)
    
    if logger.handlers:
        return logger
    
    formatter = logging.Formatter(
        "%(asctime)s [%(levelname)s] %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S"
    )
    
    # Rotate เมื่อไฟล์ > 5MB, เก็บ backup 5 ไฟล์
    rotating_handler = RotatingFileHandler(
        filename="rotating.log",
        maxBytes=5 * 1024 * 1024,  # 5 MB
        backupCount=5,              # เก็บ rotating.log.1, .2, .3, .4, .5
        encoding="utf-8"
    )
    rotating_handler.setLevel(logging.DEBUG)
    rotating_handler.setFormatter(formatter)
    
    # Console handler
    console = logging.StreamHandler()
    console.setLevel(logging.INFO)
    console.setFormatter(formatter)
    
    logger.addHandler(rotating_handler)
    logger.addHandler(console)
    
    return logger

logger = setup_rotating_logger("rotating_test")
for i in range(5):
    logger.info(f"Log message #{i+1}")
    logger.debug(f"Debug message #{i+1}: detailed info")

# TimedRotatingFileHandler: rotate ตามเวลา
def setup_timed_logger(name: str) -> logging.Logger:
    logger = logging.getLogger(name)
    logger.setLevel(logging.INFO)
    
    if logger.handlers:
        return logger
    
    formatter = logging.Formatter(
        "%(asctime)s [%(levelname)s] %(message)s"
    )
    
    # Rotate ทุกเที่ยงคืน เก็บ 30 วัน
    timed_handler = TimedRotatingFileHandler(
        filename="daily.log",
        when="midnight",     # 'S'=second, 'M'=minute, 'H'=hour, 'D'=day, 'midnight'
        interval=1,          # ทุก 1 วัน
        backupCount=30,      # เก็บ 30 ไฟล์
        encoding="utf-8"
    )
    timed_handler.setFormatter(formatter)
    logger.addHandler(timed_handler)
    
    return logger

# ทำความสะอาด
for f in ["rotating.log", "daily.log"]:
    if os.path.exists(f):
        os.remove(f)

# Logging configuration ด้วย dict (แนะนำสำหรับ production)
import logging.config

LOGGING_CONFIG = {
    "version": 1,
    "disable_existing_loggers": False,  # ไม่ disable library loggers
    
    "formatters": {
        "detailed": {
            "format": "%(asctime)s [%(levelname)-8s] %(name)s:%(lineno)d - %(message)s",
            "datefmt": "%Y-%m-%d %H:%M:%S"
        },
        "simple": {
            "format": "%(levelname)s: %(message)s"
        },
        "json": {
            "()": "pythonjsonlogger.jsonlogger.JsonFormatter",  # requires pip install
            "format": "%(asctime)s %(levelname)s %(name)s %(message)s"
        }
    },
    
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "level": "INFO",
            "formatter": "simple",
            "stream": "ext://sys.stdout"
        },
        "file": {
            "class": "logging.handlers.RotatingFileHandler",
            "level": "DEBUG",
            "formatter": "detailed",
            "filename": "app.log",
            "maxBytes": 10485760,  # 10MB
            "backupCount": 5,
            "encoding": "utf-8"
        },
        "error_file": {
            "class": "logging.FileHandler",
            "level": "ERROR",
            "formatter": "detailed",
            "filename": "errors.log",
            "encoding": "utf-8"
        }
    },
    
    "loggers": {
        "myapp": {
            "level": "DEBUG",
            "handlers": ["console", "file"],
            "propagate": False
        },
        "myapp.database": {
            "level": "INFO",
            "propagate": True  # ส่งไป myapp logger
        },
        "sqlalchemy": {
            "level": "WARNING",  # ซ่อน SQL queries ที่ verbose
            "propagate": False
        }
    },
    
    "root": {
        "level": "WARNING",
        "handlers": ["console"]
    }
}

logging.config.dictConfig(LOGGING_CONFIG)

app_logger = logging.getLogger("myapp")
app_logger.info("Application configured with dictConfig")
app_logger.debug("Debug info available in file")

# ทำความสะอาด
for f in ["app.log", "errors.log"]:
    if os.path.exists(f):
        os.remove(f)
```

---

## 7. Extra Fields และ Contextual Logging

```python
import logging
import uuid
import threading
from typing import Optional

# LoggerAdapter: เพิ่ม context ให้ทุก log message
class RequestAdapter(logging.LoggerAdapter):
    """เพิ่ม request_id ให้ทุก log"""
    
    def process(self, msg, kwargs):
        request_id = self.extra.get("request_id", "unknown")
        user_id = self.extra.get("user_id", "anonymous")
        return f"[req={request_id[:8]} user={user_id}] {msg}", kwargs

# ใช้งาน
base_logger = logging.getLogger("myapp.api")
base_logger.setLevel(logging.DEBUG)
handler = logging.StreamHandler()
handler.setFormatter(logging.Formatter("%(asctime)s [%(levelname)s] %(message)s", "%H:%M:%S"))
base_logger.addHandler(handler)

def handle_api_request(request_id: str, user_id: str, endpoint: str) -> None:
    """Simulate handling an API request"""
    # สร้าง adapter พร้อม context
    logger = RequestAdapter(base_logger, {
        "request_id": request_id,
        "user_id": user_id
    })
    
    logger.info(f"Received {endpoint}")
    logger.debug(f"Processing {endpoint} with user {user_id}")
    
    # Simulate processing
    if endpoint == "/admin":
        logger.warning(f"Admin endpoint accessed")
    
    logger.info(f"Request completed")

# ทดสอบ
handle_api_request(str(uuid.uuid4()), "user_123", "/api/products")
handle_api_request(str(uuid.uuid4()), "admin_001", "/admin")

# Thread-local context
_local = threading.local()

def set_context(request_id: str, user_id: str) -> None:
    _local.request_id = request_id
    _local.user_id = user_id

def clear_context() -> None:
    _local.request_id = None
    _local.user_id = None

class ContextualFilter(logging.Filter):
    """เพิ่ม context จาก thread-local"""
    
    def filter(self, record: logging.LogRecord) -> bool:
        record.request_id = getattr(_local, "request_id", "N/A")
        record.user_id = getattr(_local, "user_id", "anonymous")
        return True

# สร้าง logger พร้อม contextual filter
ctx_logger = logging.getLogger("contextual")
ctx_logger.setLevel(logging.DEBUG)
ctx_handler = logging.StreamHandler()
ctx_handler.setFormatter(logging.Formatter(
    "%(asctime)s [%(levelname)s] [%(request_id)s|%(user_id)s] %(message)s",
    "%H:%M:%S"
))
ctx_logger.addFilter(ContextualFilter())
ctx_logger.addHandler(ctx_handler)
ctx_logger.propagate = False

def process_request(req_id: str, uid: str) -> None:
    set_context(req_id, uid)
    ctx_logger.info("Request started")
    ctx_logger.debug("Processing data")
    ctx_logger.info("Request completed")
    clear_context()

process_request("req_abc123", "user_456")
process_request("req_xyz789", "user_789")
```

---

## 8. Exception Logging

```python
import logging
import traceback

logger = logging.getLogger("errors")
logger.setLevel(logging.DEBUG)
handler = logging.StreamHandler()
handler.setFormatter(logging.Formatter("%(levelname)s: %(message)s"))
logger.addHandler(handler)
logger.propagate = False

def divide(a: float, b: float) -> float:
    return a / b

def process_data(data: list) -> list:
    results = []
    for item in data:
        try:
            result = divide(100, item)
            results.append(result)
        except ZeroDivisionError:
            # exc_info=True บันทึก traceback
            logger.error(f"Division by zero for item={item}", exc_info=True)
        except TypeError as e:
            # logger.exception() = logger.error(..., exc_info=True)
            logger.exception(f"Type error for item={item}")
    return results

results = process_data([10, 5, 0, "abc", 2])
print(f"\nResults: {results}")

# Logging exception chain
def load_config(path: str) -> dict:
    try:
        with open(path) as f:
            import json
            return json.load(f)
    except FileNotFoundError as e:
        raise RuntimeError(f"Config file not found: {path}") from e
    except Exception as e:
        logger.error("Failed to load config", exc_info=True)
        raise

try:
    config = load_config("nonexistent.json")
except RuntimeError:
    logger.critical("Cannot start: missing configuration", exc_info=True)

# Custom exception handler (global)
import sys

def global_exception_handler(exc_type, exc_value, exc_traceback):
    """จับ uncaught exceptions"""
    if issubclass(exc_type, KeyboardInterrupt):
        sys.__excepthook__(exc_type, exc_value, exc_traceback)
        return
    
    logger.critical(
        "Uncaught exception!",
        exc_info=(exc_type, exc_value, exc_traceback)
    )

sys.excepthook = global_exception_handler
```

---

## 9. ตัวอย่างจริง: Application Logger

```python
import logging
import logging.handlers
import os
import sys
import time
import uuid
from functools import wraps
from typing import Callable, Any

class AppLogger:
    """Complete logging setup สำหรับ application"""
    
    def __init__(
        self,
        app_name: str,
        log_dir: str = "logs",
        console_level: int = logging.INFO,
        file_level: int = logging.DEBUG,
    ) -> None:
        self.app_name = app_name
        self.log_dir = log_dir
        
        # สร้าง log directory
        os.makedirs(log_dir, exist_ok=True)
        
        # Root logger สำหรับ app
        self.logger = logging.getLogger(app_name)
        self.logger.setLevel(logging.DEBUG)
        self.logger.propagate = False
        
        # ล้าง handlers เดิม
        self.logger.handlers.clear()
        
        # Formatters
        detailed_fmt = logging.Formatter(
            "%(asctime)s [%(levelname)-8s] %(name)s:%(funcName)s:%(lineno)d - %(message)s",
            datefmt="%Y-%m-%d %H:%M:%S"
        )
        simple_fmt = logging.Formatter(
            "%(asctime)s [%(levelname)s] %(message)s",
            datefmt="%H:%M:%S"
        )
        
        # Console Handler
        console = logging.StreamHandler(sys.stdout)
        console.setLevel(console_level)
        console.setFormatter(simple_fmt)
        self.logger.addHandler(console)
        
        # Rotating File Handler (แยก debug/info/warning)
        app_file = logging.handlers.RotatingFileHandler(
            filename=os.path.join(log_dir, f"{app_name}.log"),
            maxBytes=10 * 1024 * 1024,  # 10 MB
            backupCount=5,
            encoding="utf-8"
        )
        app_file.setLevel(file_level)
        app_file.setFormatter(detailed_fmt)
        self.logger.addHandler(app_file)
        
        # Error File Handler
        error_file = logging.handlers.RotatingFileHandler(
            filename=os.path.join(log_dir, f"{app_name}_errors.log"),
            maxBytes=5 * 1024 * 1024,  # 5 MB
            backupCount=10,
            encoding="utf-8"
        )
        error_file.setLevel(logging.ERROR)
        error_file.setFormatter(detailed_fmt)
        self.logger.addHandler(error_file)
    
    def get_logger(self, name: str) -> logging.Logger:
        """ดึง child logger"""
        return logging.getLogger(f"{self.app_name}.{name}")
    
    def log_function_call(self, func: Callable) -> Callable:
        """Decorator สำหรับ log function calls"""
        @wraps(func)
        def wrapper(*args, **kwargs) -> Any:
            func_logger = self.get_logger(func.__module__)
            func_name = f"{func.__qualname__}"
            
            func_logger.debug(f"CALL {func_name}(args={args[:2]}, kwargs={list(kwargs.keys())})")
            start = time.time()
            
            try:
                result = func(*args, **kwargs)
                elapsed = time.time() - start
                func_logger.debug(f"DONE {func_name} in {elapsed:.3f}s")
                return result
            except Exception as e:
                elapsed = time.time() - start
                func_logger.error(
                    f"FAIL {func_name} after {elapsed:.3f}s: {type(e).__name__}: {e}",
                    exc_info=True
                )
                raise
        
        return wrapper

# ใช้งาน
app_log = AppLogger(
    app_name="mywebapp",
    log_dir="/tmp/test_logs",
    console_level=logging.INFO
)

# ดึง loggers สำหรับแต่ละ module
db_log = app_log.get_logger("database")
api_log = app_log.get_logger("api")
auth_log = app_log.get_logger("auth")

# ทดสอบ logging
api_log.info("API server started on port 8080")
db_log.info("Connected to PostgreSQL")
auth_log.debug("JWT secret loaded")

# Decorator
@app_log.log_function_call
def authenticate_user(username: str, password: str) -> dict:
    """Authenticate user and return token"""
    # Simulate authentication
    if username == "admin" and password == "secret":
        return {"user": username, "token": str(uuid.uuid4())}
    raise ValueError(f"Invalid credentials for {username}")

@app_log.log_function_call
def fetch_products(category: str, limit: int = 10) -> list:
    """Fetch products from database"""
    time.sleep(0.05)  # Simulate DB query
    return [{"id": i, "name": f"Product {i}", "category": category} for i in range(1, limit+1)]

# ทดสอบ
try:
    user = authenticate_user("admin", "secret")
    api_log.info(f"User authenticated: {user['user']}")
    
    products = fetch_products("electronics", limit=5)
    api_log.info(f"Fetched {len(products)} products")
    
    # Test error
    authenticate_user("hacker", "wrong")
    
except ValueError as e:
    auth_log.warning(f"Authentication failed: {e}")

# ทำความสะอาด
import shutil
shutil.rmtree("/tmp/test_logs", ignore_errors=True)

# สรุปรูปแบบ logging ที่ดี
print("""
=== Best Practices for Logging ===

1. ใช้ getLogger(__name__) ไม่ใช้ root logger
   logger = logging.getLogger(__name__)

2. ไม่ใช้ print() ใน library code
   ใช้ NullHandler แทน
   logging.getLogger(__name__).addHandler(logging.NullHandler())

3. ระดับที่เหมาะสม:
   DEBUG   - ข้อมูล detailed สำหรับ debugging
   INFO    - บอกว่า flow ปกติ
   WARNING - บางอย่างผิดปกติแต่ยังทำงานได้
   ERROR   - เกิด error แต่ยังทำงานต่อได้
   CRITICAL - ระบบอาจหยุดทำงาน

4. Log exceptions ด้วย exc_info=True
   logger.error("Failed", exc_info=True)
   # หรือ
   logger.exception("Failed")  # ใน except block

5. ใช้ % formatting ไม่ใช่ f-string
   ✅ logger.debug("Value: %s", value)  # lazy evaluation
   ❌ logger.debug(f"Value: {value}")   # evaluate ทุกครั้ง

6. Structured logging สำหรับ production
   ใช้ python-json-logger หรือ structlog
""")
```

---

## 10. สรุป Part 036

✅ **5 Log Levels**: DEBUG, INFO, WARNING, ERROR, CRITICAL  
✅ **basicConfig()** ตั้งค่า root logger อย่างง่าย  
✅ **StreamHandler** output ไป console  
✅ **FileHandler** บันทึกลงไฟล์  
✅ **RotatingFileHandler** จำกัดขนาดไฟล์ด้วย rotation  
✅ **TimedRotatingFileHandler** rotate ตามเวลา  
✅ **Formatter** กำหนด log format  
✅ **getLogger(__name__)** สร้าง named logger  
✅ **Logger Hierarchy** propagation parent → child  
✅ **dictConfig** ตั้งค่าผ่าน dict (production-ready)  
✅ **exc_info=True** หรือ **logger.exception()** บันทึก traceback  

---

## ➡️ ถัดไป: Part 037 - Testing กับ pytest

*Part 036/100+ | Python Course - Beginner to World-Class*
