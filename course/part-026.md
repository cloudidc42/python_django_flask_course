# Part 026: Context Managers
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Context Manager และ with statement
- สร้าง Custom Context Manager ได้
- ใช้ contextlib module
- ประยุกต์ใช้ใน File, Database, Network connections

---

## 1. Context Manager พื้นฐาน

Context Manager จัดการ resources อัตโนมัติ (เปิด/ปิด, acquire/release)

```python
# ปัญหาที่ context manager แก้ไข
# ❌ แบบเก่า - อาจลืมปิด file
f = open("test.txt", "w")
f.write("Hello")
# ถ้า error เกิดที่นี่ → f ไม่ถูกปิด!
f.close()

# ✅ แบบ context manager - ปิดอัตโนมัติเสมอ
with open("test.txt", "w") as f:
    f.write("Hello")
    # ถ้า error เกิดที่นี่ → Python ยังปิด f ให้
# หลัง with block: f ถูกปิดแล้ว

# ตัวอย่าง context managers ที่ใช้บ่อย
# 1. File operations
with open("file.txt", "r") as f:
    content = f.read()

# 2. Multiple context managers
with open("input.txt") as fin, open("output.txt", "w") as fout:
    fout.write(fin.read().upper())

# 3. Database connections
import sqlite3
with sqlite3.connect("mydb.db") as conn:
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users")
    results = cursor.fetchall()

# 4. Lock (threading)
import threading
lock = threading.Lock()
with lock:
    # critical section - thread-safe
    pass
```

---

## 2. สร้าง Context Manager ด้วย Class

```python
class FileManager:
    """Custom context manager สำหรับ file"""
    
    def __init__(self, filename, mode="r"):
        self.filename = filename
        self.mode = mode
        self.file = None
    
    def __enter__(self):
        """เรียกเมื่อเข้า with block"""
        print(f"Opening {self.filename}")
        self.file = open(self.filename, self.mode)
        return self.file  # ค่าที่ถูก assign ให้ 'as' variable
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        """เรียกเมื่อออกจาก with block"""
        print(f"Closing {self.filename}")
        if self.file:
            self.file.close()
        
        # Return True เพื่อ suppress exception
        # Return None/False เพื่อ propagate exception
        if exc_type is not None:
            print(f"Exception occurred: {exc_type.__name__}: {exc_val}")
        return False  # propagate exception

# ใช้งาน
with FileManager("test.txt", "w") as f:
    f.write("Hello from context manager!")

with FileManager("test.txt", "r") as f:
    content = f.read()
    print(content)

# Timer context manager
import time

class Timer:
    """วัดเวลาการทำงาน"""
    
    def __init__(self, name=""):
        self.name = name
    
    def __enter__(self):
        self.start = time.perf_counter()
        return self
    
    def __exit__(self, *args):
        self.elapsed = time.perf_counter() - self.start
        name = f"[{self.name}] " if self.name else ""
        print(f"{name}Elapsed: {self.elapsed:.4f}s")
        return False

with Timer("Sorting"):
    data = list(range(1000000, 0, -1))
    data.sort()

# Database Transaction Manager
class DatabaseTransaction:
    """จัดการ database transaction"""
    
    def __init__(self, connection):
        self.conn = connection
    
    def __enter__(self):
        self.conn.execute("BEGIN")
        return self.conn.cursor()
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is None:
            self.conn.execute("COMMIT")
            print("Transaction committed")
        else:
            self.conn.execute("ROLLBACK")
            print(f"Transaction rolled back: {exc_val}")
        return False
```

---

## 3. สร้าง Context Manager ด้วย contextlib

```python
from contextlib import contextmanager, asynccontextmanager

@contextmanager
def timer(name=""):
    """Context manager ด้วย generator"""
    start = time.perf_counter()
    try:
        yield  # ส่วน with block ทำงานที่นี่
    finally:
        elapsed = time.perf_counter() - start
        label = f"[{name}] " if name else ""
        print(f"{label}Elapsed: {elapsed:.4f}s")

with timer("My Operation"):
    result = sum(range(1_000_000))

@contextmanager
def managed_resource(name):
    """จัดการ resource ทั่วไป"""
    print(f"Acquiring {name}")
    resource = {"name": name, "acquired": True}
    try:
        yield resource
    except Exception as e:
        print(f"Error while using {name}: {e}")
        raise
    finally:
        print(f"Releasing {name}")
        resource["acquired"] = False

with managed_resource("Database Connection") as conn:
    print(f"Using {conn['name']}")
    # เพิ่มโค้ดที่นี่

# Suppress specific exceptions
from contextlib import suppress

with suppress(FileNotFoundError):
    with open("nonexistent.txt") as f:
        data = f.read()
# ไม่ error แม้ file ไม่มี

# redirect_stdout
from contextlib import redirect_stdout
import io

buffer = io.StringIO()
with redirect_stdout(buffer):
    print("This goes to buffer")
    print("Not to console")

output = buffer.getvalue()
print(f"Captured: {output!r}")

# ExitStack - จัดการ context managers แบบ dynamic
from contextlib import ExitStack

filenames = ["file1.txt", "file2.txt", "file3.txt"]
with ExitStack() as stack:
    files = [stack.enter_context(open(f, "w")) for f in filenames]
    for i, f in enumerate(files):
        f.write(f"Content {i}")
```

---

## 4. Async Context Managers

```python
import asyncio
from contextlib import asynccontextmanager

class AsyncDatabase:
    """Async database connection manager"""
    
    async def __aenter__(self):
        print("Connecting to database...")
        await asyncio.sleep(0.1)  # simulate async connection
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        print("Closing database connection...")
        await asyncio.sleep(0.05)  # simulate cleanup
        return False
    
    async def query(self, sql):
        await asyncio.sleep(0.1)  # simulate query
        return [{"id": 1, "name": "Alice"}]

async def main():
    async with AsyncDatabase() as db:
        results = await db.query("SELECT * FROM users")
        print(results)

asyncio.run(main())

@asynccontextmanager
async def async_timer(name=""):
    start = asyncio.get_event_loop().time()
    try:
        yield
    finally:
        elapsed = asyncio.get_event_loop().time() - start
        print(f"[{name}] Async elapsed: {elapsed:.4f}s")

async def demo():
    async with async_timer("Async Operation"):
        await asyncio.sleep(0.5)

asyncio.run(demo())
```

---

## 5. ตัวอย่างโปรแกรม: Resource Manager

```python
# resource_manager.py
"""
ระบบจัดการ resources ด้วย context managers
"""
import os
import time
import logging
from contextlib import contextmanager

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class ConnectionPool:
    """จำลอง connection pool"""
    
    def __init__(self, max_connections=5):
        self.max_connections = max_connections
        self.active_connections = 0
        self._connections = []
    
    def get_connection(self):
        if self.active_connections >= self.max_connections:
            raise RuntimeError("No available connections")
        
        conn = {"id": self.active_connections + 1, "active": True}
        self.active_connections += 1
        self._connections.append(conn)
        logger.info(f"Connection {conn['id']} acquired ({self.active_connections}/{self.max_connections})")
        return conn
    
    def release_connection(self, conn):
        conn["active"] = False
        self.active_connections -= 1
        logger.info(f"Connection {conn['id']} released ({self.active_connections}/{self.max_connections})")

pool = ConnectionPool(max_connections=3)

@contextmanager
def get_db_connection():
    """Context manager สำหรับ database connection จาก pool"""
    conn = None
    try:
        conn = pool.get_connection()
        yield conn
    except RuntimeError as e:
        logger.error(f"Failed to get connection: {e}")
        raise
    finally:
        if conn and conn.get("active"):
            pool.release_connection(conn)

# ใช้งาน
def process_user(user_id):
    with get_db_connection() as conn:
        logger.info(f"Processing user {user_id} with conn {conn['id']}")
        time.sleep(0.01)  # simulate work
        return {"user_id": user_id, "status": "processed"}

# Process หลาย users
for i in range(1, 5):
    result = process_user(i)
    print(result)
```

---

## 6. สรุป Part 026

✅ **with statement** - จัดการ resources อัตโนมัติ  
✅ **__enter__ / __exit__** - protocol ของ context manager  
✅ **@contextmanager** - สร้างด้วย generator function  
✅ **contextlib** - suppress, redirect_stdout, ExitStack  
✅ **Async Context Managers** - async with  

---

## ➡️ ถัดไป: Part 027 - Type Hints และ Annotations

*Part 026/100+ | Python Course - Beginner to World-Class*
