# Part 012: File I/O
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- เปิด อ่าน เขียน และปิดไฟล์ได้
- ใช้ Context Manager กับไฟล์ได้
- รู้จัก file modes ต่างๆ
- อ่านและเขียน CSV files ได้
- อ่านและเขียน JSON files ได้
- ใช้ pathlib และ os.path ได้
- จัดการ file operations ต่างๆ ได้

---

## 1. การเปิดและปิดไฟล์

```python
# ===== วิธีที่ 1: open() และ close() (แบบเก่า) =====
# ⚠️ ต้องระวังเรื่อง close() เสมอ!

f = open("example.txt", "w", encoding="utf-8")
try:
    f.write("Hello, World!\n")
    f.write("สวัสดีชาวโลก\n")
finally:
    f.close()  # ต้อง close เสมอ ไม่งั้น resource leak!

# ===== วิธีที่ 2: Context Manager (แนะนำ!) =====
# with statement จะ close ไฟล์ให้อัตโนมัติ แม้เกิด error

with open("example.txt", "w", encoding="utf-8") as f:
    f.write("Hello, World!\n")
    f.write("สวัสดีชาวโลก\n")
# f ถูก close อัตโนมัติตรงนี้

# เปิดหลายไฟล์พร้อมกัน
with open("input.txt", "r", encoding="utf-8") as fin, \
     open("output.txt", "w", encoding="utf-8") as fout:
    content = fin.read()
    fout.write(content.upper())

print("เขียนไฟล์เสร็จแล้ว")
```

## 2. File Modes

```python
"""
File Modes:
"r"  - read only (default) - error ถ้าไม่มีไฟล์
"w"  - write only - สร้างใหม่หรือล้างไฟล์เดิม
"a"  - append - เพิ่มต่อท้าย, สร้างใหม่ถ้าไม่มี
"x"  - exclusive create - error ถ้ามีไฟล์อยู่แล้ว
"r+" - read and write - error ถ้าไม่มีไฟล์
"w+" - read and write - สร้างใหม่หรือล้าง
"a+" - read and append

เพิ่ม "b" สำหรับ binary mode:
"rb", "wb", "ab", "rb+", "wb+", "ab+"
"""

import os

# ===== "w" - Write Mode =====
with open("/tmp/demo_write.txt", "w", encoding="utf-8") as f:
    f.write("บรรทัดที่ 1\n")
    f.write("บรรทัดที่ 2\n")
print("เขียน demo_write.txt แล้ว")

# ===== "r" - Read Mode =====
with open("/tmp/demo_write.txt", "r", encoding="utf-8") as f:
    content = f.read()
print(f"อ่านได้: {repr(content)}")

# ===== "a" - Append Mode =====
with open("/tmp/demo_write.txt", "a", encoding="utf-8") as f:
    f.write("บรรทัดที่ 3\n")  # เพิ่มต่อท้าย ไม่ล้างของเดิม

# ===== "x" - Exclusive Mode =====
try:
    with open("/tmp/demo_write.txt", "x", encoding="utf-8") as f:
        f.write("นี่จะ error เพราะไฟล์มีอยู่แล้ว")
except FileExistsError:
    print("⚠️  FileExistsError: ไฟล์มีอยู่แล้ว")

# ===== Binary Mode =====
# อ่านรูปภาพหรือไฟล์ binary
# with open("image.png", "rb") as f:
#     data = f.read()
#     print(f"ขนาดไฟล์: {len(data)} bytes")
```

## 3. การอ่านไฟล์

```python
# สร้างไฟล์ตัวอย่างก่อน
sample_text = """Python is a programming language.
Python is easy to learn.
Python is powerful.
Python is popular.
"""

with open("/tmp/sample.txt", "w", encoding="utf-8") as f:
    f.write(sample_text)

# ===== .read() - อ่านทั้งหมด =====
with open("/tmp/sample.txt", "r", encoding="utf-8") as f:
    content = f.read()
print(f"ทั้งหมด:\n{content}")

# ===== .read(n) - อ่านกี่ chars =====
with open("/tmp/sample.txt", "r", encoding="utf-8") as f:
    first_10 = f.read(10)   # อ่าน 10 chars
    next_10 = f.read(10)    # อ่านต่อ
print(f"10 ตัวแรก: '{first_10}'")
print(f"10 ตัวถัดไป: '{next_10}'")

# ===== .readline() - อ่านทีละบรรทัด =====
with open("/tmp/sample.txt", "r", encoding="utf-8") as f:
    line1 = f.readline()
    line2 = f.readline()
print(f"บรรทัด 1: {repr(line1)}")
print(f"บรรทัด 2: {repr(line2)}")

# ===== .readlines() - อ่านทุกบรรทัดเป็น list =====
with open("/tmp/sample.txt", "r", encoding="utf-8") as f:
    lines = f.readlines()
print(f"จำนวนบรรทัด: {len(lines)}")
print(f"บรรทัดที่ 1: {repr(lines[0])}")

# ===== วนลูป (แนะนำ - ประหยัด memory) =====
with open("/tmp/sample.txt", "r", encoding="utf-8") as f:
    for i, line in enumerate(f, 1):
        print(f"{i}: {line.rstrip()}")

# ===== .seek() และ .tell() =====
with open("/tmp/sample.txt", "r", encoding="utf-8") as f:
    pos = f.tell()          # ตำแหน่งปัจจุบัน
    print(f"ตำแหน่งเริ่ม: {pos}")  # 0
    
    f.read(10)
    pos = f.tell()
    print(f"หลังอ่าน 10: {pos}")  # 10
    
    f.seek(0)              # กลับไปต้น
    first = f.read(6)
    print(f"อ่านจากต้น: '{first}'")

# ===== อ่านไฟล์ใหญ่ทีละ chunk =====
def read_in_chunks(filename: str, chunk_size: int = 4096):
    """อ่านไฟล์ใหญ่ทีละ chunk เพื่อประหยัด memory"""
    with open(filename, "r", encoding="utf-8") as f:
        while True:
            chunk = f.read(chunk_size)
            if not chunk:
                break
            yield chunk

total_chars = sum(len(chunk) for chunk in read_in_chunks("/tmp/sample.txt"))
print(f"ตัวอักษรทั้งหมด: {total_chars}")
```

## 4. การเขียนไฟล์

```python
# ===== .write() =====
with open("/tmp/output.txt", "w", encoding="utf-8") as f:
    f.write("บรรทัดที่ 1\n")
    f.write("บรรทัดที่ 2\n")
    
    # write ไม่เพิ่ม newline อัตโนมัติ!
    chars_written = f.write("บรรทัดที่ 3\n")  # return จำนวน chars
    print(f"เขียน {chars_written} ตัวอักษร")

# ===== .writelines() =====
lines = ["apple\n", "banana\n", "cherry\n"]
with open("/tmp/fruits.txt", "w", encoding="utf-8") as f:
    f.writelines(lines)  # ไม่เพิ่ม newline อัตโนมัติ!

# ===== print() กับ file argument =====
with open("/tmp/print_output.txt", "w", encoding="utf-8") as f:
    print("Hello from print!", file=f)
    print("Line 2", file=f)
    print("Numbers:", 1, 2, 3, sep=", ", file=f)

# ===== เขียนข้อมูลจาก list =====
students = [
    {"name": "Alice", "score": 95},
    {"name": "Bob", "score": 87},
    {"name": "Charlie", "score": 78},
]

with open("/tmp/students.txt", "w", encoding="utf-8") as f:
    f.write("รายชื่อนักเรียน\n")
    f.write("=" * 30 + "\n")
    for student in students:
        f.write(f"{student['name']:15} {student['score']}\n")

print("เขียนไฟล์นักเรียนแล้ว")
```

## 5. CSV Files

```python
import csv

# ===== เขียน CSV =====
students = [
    ["Alice", "A", 95, "ผ่าน"],
    ["Bob", "B", 87, "ผ่าน"],
    ["Charlie", "C", 65, "ผ่าน"],
    ["Diana", "F", 45, "ไม่ผ่าน"],
]

headers = ["ชื่อ", "เกรด", "คะแนน", "ผล"]

with open("/tmp/students.csv", "w", newline="", encoding="utf-8-sig") as f:
    writer = csv.writer(f)
    writer.writerow(headers)        # เขียน headers
    writer.writerows(students)      # เขียนทุกแถว

# ===== อ่าน CSV =====
with open("/tmp/students.csv", "r", encoding="utf-8-sig") as f:
    reader = csv.reader(f)
    headers = next(reader)  # อ่าน headers
    print(f"Headers: {headers}")
    
    for row in reader:
        print(f"  {row}")

# ===== DictWriter / DictReader =====
products = [
    {"id": "P001", "name": "Python Book", "price": 599, "stock": 50},
    {"id": "P002", "name": "USB Hub", "price": 890, "stock": 30},
    {"id": "P003", "name": "Keyboard", "price": 3500, "stock": 10},
]

# เขียนด้วย DictWriter
with open("/tmp/products.csv", "w", newline="", encoding="utf-8-sig") as f:
    fieldnames = ["id", "name", "price", "stock"]
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    
    writer.writeheader()    # เขียน headers อัตโนมัติ
    writer.writerows(products)

# อ่านด้วย DictReader
with open("/tmp/products.csv", "r", encoding="utf-8-sig") as f:
    reader = csv.DictReader(f)
    print("\nสินค้าทั้งหมด:")
    for row in reader:
        print(f"  {row['name']}: {int(row['price']):,} บาท (stock: {row['stock']})")

# ===== จัดการ CSV ขั้นสูง =====
import csv
import io

# อ่าน CSV จาก string (มีประโยชน์กับ API response)
csv_data = """name,age,city
Alice,30,Bangkok
Bob,25,Chiang Mai
Charlie,35,Phuket"""

reader = csv.DictReader(io.StringIO(csv_data))
people = list(reader)
print(f"\nคนที่อายุน้อยกว่า 30:")
for person in people:
    if int(person["age"]) < 30:
        print(f"  {person['name']} ({person['city']})")
```

## 6. JSON Files

```python
import json

# ===== เขียน JSON =====
config = {
    "app_name": "My Python App",
    "version": "1.0.0",
    "database": {
        "host": "localhost",
        "port": 5432,
        "name": "myapp_db"
    },
    "features": ["auth", "notifications", "reports"],
    "debug": False,
    "max_connections": 100
}

# json.dump() - เขียนลงไฟล์
with open("/tmp/config.json", "w", encoding="utf-8") as f:
    json.dump(config, f, indent=2, ensure_ascii=False)

print("เขียน config.json แล้ว")

# ===== อ่าน JSON =====
with open("/tmp/config.json", "r", encoding="utf-8") as f:
    loaded_config = json.load(f)

print(f"App: {loaded_config['app_name']}")
print(f"DB Host: {loaded_config['database']['host']}")
print(f"Features: {loaded_config['features']}")

# ===== JSON String =====
# json.dumps() - แปลงเป็น string
config_str = json.dumps(config, indent=2, ensure_ascii=False)
print(f"\nJSON string:\n{config_str[:200]}...")

# json.loads() - แปลง string กลับเป็น dict
data = json.loads(config_str)
print(f"\nLoaded from string: {data['version']}")

# ===== JSON กับ Thai text =====
thai_data = {
    "ชื่อ": "สมชาย ใจดี",
    "ที่อยู่": "กรุงเทพมหานคร",
    "อายุ": 30
}

# ensure_ascii=False เพื่อเก็บ Thai characters
json_str = json.dumps(thai_data, ensure_ascii=False, indent=2)
print(f"\nThai JSON:\n{json_str}")

# ===== Custom JSON Encoder =====
from datetime import datetime, date
from decimal import Decimal

class CustomEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, (datetime, date)):
            return obj.isoformat()
        if isinstance(obj, Decimal):
            return float(obj)
        if isinstance(obj, set):
            return list(obj)
        return super().default(obj)

complex_data = {
    "timestamp": datetime.now(),
    "date": date.today(),
    "price": Decimal("99.99"),
    "tags": {"python", "django", "flask"}
}

json_str = json.dumps(complex_data, cls=CustomEncoder, indent=2)
print(f"\nComplex JSON:\n{json_str}")

# ===== ตัวอย่าง: JSON API Response =====
def save_api_response(filename: str, data: dict) -> None:
    """บันทึก API response ลงไฟล์"""
    with open(filename, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)
    print(f"บันทึกแล้ว: {filename}")

def load_api_response(filename: str) -> dict:
    """โหลด API response จากไฟล์"""
    try:
        with open(filename, "r", encoding="utf-8") as f:
            return json.load(f)
    except FileNotFoundError:
        return {}
    except json.JSONDecodeError as e:
        print(f"JSON Error: {e}")
        return {}
```

## 7. pathlib - จัดการ Paths แบบทันสมัย

```python
from pathlib import Path

# ===== สร้าง Path objects =====
# Path object แทน string paths
home = Path.home()            # /home/user
cwd = Path.cwd()              # current working directory
tmp = Path("/tmp")

print(f"Home: {home}")
print(f"CWD: {cwd}")

# สร้าง path ด้วย /
data_dir = tmp / "course_data"
file_path = data_dir / "test.txt"

print(f"Data dir: {data_dir}")
print(f"File path: {file_path}")

# ===== Path Properties =====
p = Path("/tmp/course_data/notes.txt")

print(f"name: {p.name}")           # notes.txt
print(f"stem: {p.stem}")           # notes
print(f"suffix: {p.suffix}")       # .txt
print(f"suffixes: {p.suffixes}")   # ['.txt']
print(f"parent: {p.parent}")       # /tmp/course_data
print(f"parents: {list(p.parents)}")  # [Path('/tmp/course_data'), Path('/tmp'), Path('/')]
print(f"parts: {p.parts}")         # ('/', 'tmp', 'course_data', 'notes.txt')

# ===== เช็คสถานะ =====
tmp_path = Path("/tmp")
fake_path = Path("/tmp/nonexistent_file_12345.txt")

print(f"\n/tmp exists: {tmp_path.exists()}")
print(f"fake exists: {fake_path.exists()}")
print(f"/tmp is_dir: {tmp_path.is_dir()}")
print(f"/tmp is_file: {tmp_path.is_file()}")

# ===== สร้าง directories =====
new_dir = Path("/tmp/python_course/lesson1")
new_dir.mkdir(parents=True, exist_ok=True)  # สร้างทั้ง path
print(f"สร้าง directory: {new_dir}")

# ===== อ่าน/เขียนไฟล์ =====
file_path = new_dir / "notes.txt"

# เขียน
file_path.write_text("Hello from pathlib!\nบันทึกการเรียน Python", encoding="utf-8")

# อ่าน
content = file_path.read_text(encoding="utf-8")
print(f"\nอ่านจาก pathlib:\n{content}")

# อ่าน/เขียน binary
# file_path.write_bytes(b"\x00\x01\x02\x03")
# data = file_path.read_bytes()

# ===== glob - ค้นหาไฟล์ =====
# หาไฟล์ .txt ใน /tmp
txt_files = list(Path("/tmp").glob("*.txt"))
print(f"\n.txt files ใน /tmp: {len(txt_files)} ไฟล์")

# หา recursive
all_txt = list(Path("/tmp").rglob("*.txt"))
print(f".txt files ทั้งหมด: {len(all_txt)} ไฟล์")

# ===== ข้อมูลไฟล์ =====
stat = file_path.stat()
print(f"\nข้อมูลไฟล์ {file_path.name}:")
print(f"  ขนาด: {stat.st_size} bytes")

from datetime import datetime
print(f"  แก้ไขล่าสุด: {datetime.fromtimestamp(stat.st_mtime)}")

# ===== rename และ copy =====
import shutil

# copy ไฟล์
copy_path = new_dir / "notes_backup.txt"
shutil.copy2(file_path, copy_path)
print(f"\nCopy ไปที่: {copy_path}")

# rename
renamed = new_dir / "notes_v2.txt"
copy_path.rename(renamed)
print(f"Rename เป็น: {renamed}")
```

## 8. os.path - แบบดั้งเดิม

```python
import os
import os.path

# ===== Path operations =====
path = "/tmp/python_course/lesson1/notes.txt"

print(f"dirname: {os.path.dirname(path)}")   # /tmp/python_course/lesson1
print(f"basename: {os.path.basename(path)}") # notes.txt
print(f"splitext: {os.path.splitext(path)}") # ('/tmp/.../notes', '.txt')
print(f"split: {os.path.split(path)}")       # ('/tmp/.../lesson1', 'notes.txt')

# ===== Join paths =====
joined = os.path.join("/home", "user", "documents", "file.txt")
print(f"joined: {joined}")

# ===== ตรวจสอบ =====
print(f"exists: {os.path.exists(path)}")
print(f"isfile: {os.path.isfile(path)}")
print(f"isdir: {os.path.isdir('/tmp')}")

# ===== ขนาดไฟล์ =====
if os.path.exists(path):
    size = os.path.getsize(path)
    print(f"ขนาด: {size} bytes")

# ===== expanduser: ~ =====
home_file = os.path.expanduser("~/documents/notes.txt")
print(f"expand ~: {home_file}")

# ===== abspath =====
abs_path = os.path.abspath("./relative/path.txt")
print(f"absolute: {abs_path}")

# ===== listdir =====
files = os.listdir("/tmp")
print(f"\nไฟล์ใน /tmp ({len(files)} รายการ):")
for f in sorted(files)[:5]:
    print(f"  {f}")

# ===== walk - วนทุก directory =====
for root, dirs, files in os.walk("/tmp/python_course"):
    level = root.replace("/tmp/python_course", "").count(os.sep)
    indent = "  " * level
    print(f"{indent}{os.path.basename(root)}/")
    subindent = "  " * (level + 1)
    for file in files:
        print(f"{subindent}{file}")
```

## 9. File Operations ขั้นสูง

```python
import shutil
import os
from pathlib import Path

# ===== Copy, Move, Delete =====

# ตัวอย่าง paths
src = Path("/tmp/python_course/lesson1/notes.txt")
dst_dir = Path("/tmp/python_course/backup")
dst_dir.mkdir(parents=True, exist_ok=True)

# copy ไฟล์ (preserve metadata)
dst = shutil.copy2(src, dst_dir)
print(f"Copied to: {dst}")

# copy directory
# shutil.copytree("/tmp/python_course/lesson1", "/tmp/backup_lesson1")

# move ไฟล์
# shutil.move(src, dst_dir / "notes_moved.txt")

# ลบไฟล์
temp = dst_dir / "temp_file.txt"
temp.write_text("temp content")
temp.unlink()  # ลบไฟล์
print("ลบไฟล์ temp แล้ว")

# ลบ directory
# shutil.rmtree("/tmp/old_directory")  # ⚠️ ระวัง ลบทุกอย่าง!

# ===== Temporary Files =====
import tempfile

# สร้าง temp file
with tempfile.NamedTemporaryFile(mode="w", suffix=".txt", 
                                  delete=False, encoding="utf-8") as tmp:
    tmp.write("Temporary content")
    tmp_path = tmp.name

print(f"Temp file: {tmp_path}")
# ทำงานกับไฟล์...
os.unlink(tmp_path)  # ลบเมื่อเสร็จ

# สร้าง temp directory
with tempfile.TemporaryDirectory() as tmpdir:
    print(f"Temp dir: {tmpdir}")
    # ทำงานใน tmpdir...
    # จะถูกลบอัตโนมัติเมื่อออกจาก with block

# ===== File Locking (สำหรับ concurrent access) =====
import fcntl
import time

def read_with_lock(filename: str) -> str:
    """อ่านไฟล์พร้อม lock"""
    with open(filename, "r", encoding="utf-8") as f:
        fcntl.flock(f, fcntl.LOCK_SH)  # shared lock
        try:
            return f.read()
        finally:
            fcntl.flock(f, fcntl.LOCK_UN)  # unlock

# ===== ตัวอย่าง: Log File Writer =====
from datetime import datetime

class LogWriter:
    """เขียน log file พร้อม rotation"""
    
    def __init__(self, log_dir: str, prefix: str = "app"):
        self.log_dir = Path(log_dir)
        self.log_dir.mkdir(parents=True, exist_ok=True)
        self.prefix = prefix
    
    def _get_log_path(self) -> Path:
        today = datetime.now().strftime("%Y-%m-%d")
        return self.log_dir / f"{self.prefix}_{today}.log"
    
    def log(self, level: str, message: str) -> None:
        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        log_entry = f"[{timestamp}] [{level:8}] {message}\n"
        
        with open(self._get_log_path(), "a", encoding="utf-8") as f:
            f.write(log_entry)
    
    def info(self, msg: str) -> None:
        self.log("INFO", msg)
    
    def error(self, msg: str) -> None:
        self.log("ERROR", msg)
    
    def warning(self, msg: str) -> None:
        self.log("WARNING", msg)
    
    def read_today(self) -> str:
        log_path = self._get_log_path()
        if log_path.exists():
            return log_path.read_text(encoding="utf-8")
        return ""
    
    def get_log_files(self) -> list:
        """หาไฟล์ log ทั้งหมด"""
        return sorted(self.log_dir.glob(f"{self.prefix}_*.log"))

# ===== ทดสอบ LogWriter =====
logger = LogWriter("/tmp/app_logs", "myapp")

logger.info("Application started")
logger.info("User logged in: alice@example.com")
logger.warning("High memory usage: 85%")
logger.error("Database connection failed")
logger.info("Retrying database connection...")
logger.info("Database connected successfully")

print("\nLog files:")
for log_file in logger.get_log_files():
    print(f"  {log_file.name}")

print("\nLog content today:")
print(logger.read_today())
```

## 10. ตัวอย่างโปรเจกต์: Config Manager

```python
"""
Configuration Manager
- อ่าน/เขียน config จาก JSON
- รองรับ environment variables
- validation
"""

import json
import os
from pathlib import Path
from typing import Any

class ConfigManager:
    """จัดการ configuration ของแอป"""
    
    DEFAULT_CONFIG = {
        "app": {
            "name": "My App",
            "version": "1.0.0",
            "debug": False
        },
        "database": {
            "host": "localhost",
            "port": 5432,
            "name": "myapp",
            "pool_size": 10
        },
        "cache": {
            "backend": "redis",
            "host": "localhost",
            "port": 6379,
            "ttl": 300
        },
        "logging": {
            "level": "INFO",
            "file": "/tmp/app.log"
        }
    }
    
    def __init__(self, config_path: str = "/tmp/app_config.json"):
        self.config_path = Path(config_path)
        self._config = {}
        self._load()
    
    def _load(self) -> None:
        """โหลด config จากไฟล์"""
        if self.config_path.exists():
            with open(self.config_path, "r", encoding="utf-8") as f:
                self._config = json.load(f)
        else:
            self._config = self.DEFAULT_CONFIG.copy()
            self._save()
    
    def _save(self) -> None:
        """บันทึก config ลงไฟล์"""
        self.config_path.parent.mkdir(parents=True, exist_ok=True)
        with open(self.config_path, "w", encoding="utf-8") as f:
            json.dump(self._config, f, indent=2, ensure_ascii=False)
    
    def get(self, key: str, default: Any = None) -> Any:
        """ดึงค่า config ด้วย dot notation เช่น 'database.host'"""
        keys = key.split(".")
        value = self._config
        
        for k in keys:
            if isinstance(value, dict):
                value = value.get(k)
            else:
                return default
            
            if value is None:
                return default
        
        return value
    
    def set(self, key: str, value: Any) -> None:
        """ตั้งค่า config ด้วย dot notation"""
        keys = key.split(".")
        config = self._config
        
        for k in keys[:-1]:
            if k not in config:
                config[k] = {}
            config = config[k]
        
        config[keys[-1]] = value
        self._save()
    
    def get_with_env(self, key: str, env_var: str = None, default: Any = None) -> Any:
        """ดึงค่าจาก env variable ก่อน แล้วค่อย fallback ไป config"""
        if env_var:
            env_val = os.getenv(env_var)
            if env_val is not None:
                return env_val
        return self.get(key, default)
    
    def show(self) -> None:
        """แสดง config ทั้งหมด"""
        print(json.dumps(self._config, indent=2, ensure_ascii=False))

# ===== ทดสอบ ConfigManager =====
config = ConfigManager("/tmp/demo_config.json")

print("Config เริ่มต้น:")
config.show()

# ดึงค่า
print(f"\nApp name: {config.get('app.name')}")
print(f"DB host: {config.get('database.host')}")
print(f"Debug: {config.get('app.debug')}")
print(f"Not exist: {config.get('not.exist', 'default_value')}")

# ตั้งค่า
config.set("app.debug", True)
config.set("database.host", "prod-db.example.com")
config.set("app.new_feature", {"enabled": True, "limit": 100})

print("\nConfig หลังแก้ไข:")
config.show()

# ดึงค่าจาก env variable
db_password = config.get_with_env(
    "database.password", 
    env_var="DB_PASSWORD", 
    default="secret123"
)
print(f"\nDB Password: {db_password}")
```

## 11. สรุป Part 012

ใน Part นี้คุณได้เรียนรู้:

✅ **การเปิด/ปิดไฟล์** - `open()`, `close()`, context manager (`with`)
✅ **File Modes** - `r`, `w`, `a`, `x`, `r+`, `w+`, `a+`, `b` modes
✅ **การอ่านไฟล์** - `read()`, `readline()`, `readlines()`, loop, `seek()`, `tell()`
✅ **การเขียนไฟล์** - `write()`, `writelines()`, `print(file=f)`
✅ **CSV Files** - `csv.reader`, `csv.writer`, `DictReader`, `DictWriter`
✅ **JSON Files** - `json.load()`, `json.dump()`, `json.loads()`, `json.dumps()`, Custom Encoder
✅ **pathlib** - Path objects, `mkdir()`, `glob()`, `rglob()`, `read_text()`, `write_text()`
✅ **os.path** - `dirname()`, `basename()`, `join()`, `exists()`, `walk()`
✅ **File Operations** - `shutil.copy2()`, `shutil.move()`, tempfile

---

## ➡️ ถัดไป: Part 013 - Exception Handling

*Part 012/100+ | Python Course - Beginner to World-Class*
