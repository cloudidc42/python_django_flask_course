# Part 012: File I/O
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- อ่านและเขียนไฟล์ text และ binary ได้
- ใช้ Context Manager (with statement) ได้
- ทำงานกับ CSV files ด้วย csv module ได้
- ทำงานกับ JSON files ได้
- ใช้ pathlib สำหรับ file/directory operations ได้

---

## 1. การเปิดและปิดไฟล์

```python
# open() - เปิดไฟล์
# Syntax: open(file, mode='r', encoding=None, ...)
# Modes: 'r' read, 'w' write, 'a' append, 'x' exclusive create
#        'b' binary, 't' text (default), '+' read+write

# วิธีที่ 1: แบบ manual (ไม่แนะนำ)
f = open("example.txt", "w", encoding="utf-8")
f.write("Hello, World!\n")
f.write("สวัสดีโลก\n")
f.close()  # ⚠️ ต้องปิดเสมอ มิเช่นนั้น data อาจสูญหาย

# วิธีที่ 2: with statement (แนะนำ) - ปิดอัตโนมัติ
with open("example.txt", "w", encoding="utf-8") as f:
    f.write("Hello, World!\n")
    f.write("สวัสดีโลก\n")
# ไฟล์ปิดอัตโนมัติเมื่อออกจาก with block

# เปิดหลายไฟล์พร้อมกัน
with open("input.txt", "r") as fin, open("output.txt", "w") as fout:
    content = fin.read()
    fout.write(content.upper())
```

---

## 2. การอ่านไฟล์

```python
# สร้างไฟล์ทดสอบ
with open("test.txt", "w", encoding="utf-8") as f:
    f.write("Line 1: Hello World\n")
    f.write("Line 2: Python Programming\n")
    f.write("Line 3: File I/O Example\n")
    f.write("Line 4: สวัสดีโลก\n")
    f.write("Line 5: The End\n")

# read() - อ่านทั้งหมดเป็น string
with open("test.txt", "r", encoding="utf-8") as f:
    content = f.read()
    print(content)
    print(f"ขนาด: {len(content)} chars")

# read(n) - อ่าน n characters
with open("test.txt", "r", encoding="utf-8") as f:
    first_10 = f.read(10)
    print(f"10 chars แรก: {first_10!r}")
    
    next_10 = f.read(10)
    print(f"10 chars ถัดไป: {next_10!r}")

# readline() - อ่านทีละบรรทัด
with open("test.txt", "r", encoding="utf-8") as f:
    line1 = f.readline()    # อ่าน 1 บรรทัด (รวม \n)
    line2 = f.readline()
    print(f"บรรทัด 1: {line1!r}")
    print(f"บรรทัด 2: {line2!r}")

# readlines() - อ่านทั้งหมดเป็น list
with open("test.txt", "r", encoding="utf-8") as f:
    lines = f.readlines()   # ['Line 1: ...\n', 'Line 2: ...\n', ...]
    print(f"จำนวนบรรทัด: {len(lines)}")
    for i, line in enumerate(lines, 1):
        print(f"  [{i}] {line.rstrip()}")

# Iterate ทีละบรรทัด (ดีสุดสำหรับไฟล์ใหญ่ - ไม่โหลดทั้งหมดลง memory)
with open("test.txt", "r", encoding="utf-8") as f:
    for line in f:
        print(line.rstrip())

# tell() - ตำแหน่งปัจจุบัน
# seek() - ย้ายไปตำแหน่งที่กำหนด
with open("test.txt", "r", encoding="utf-8") as f:
    print(f"เริ่มต้น: {f.tell()}")
    f.read(10)
    print(f"หลังอ่าน 10: {f.tell()}")
    f.seek(0)               # กลับต้นไฟล์
    print(f"หลัง seek(0): {f.tell()}")
    f.seek(0, 2)            # ไปท้ายไฟล์ (2=SEEK_END)
    print(f"ท้ายไฟล์: {f.tell()}")
```

---

## 3. การเขียนไฟล์

```python
# write() - เขียน string (ไม่มี auto newline)
with open("output.txt", "w", encoding="utf-8") as f:
    f.write("Line 1\n")
    f.write("Line 2\n")
    chars_written = f.write("Line 3\n")  # คืนจำนวน chars ที่เขียน
    print(f"เขียน {chars_written} chars")

# writelines() - เขียน list ของ strings (ไม่มี auto newline)
lines = ["apple\n", "banana\n", "cherry\n"]
with open("fruits.txt", "w", encoding="utf-8") as f:
    f.writelines(lines)

# append mode - เพิ่มท้ายไฟล์ (ไม่ลบข้อมูลเดิม)
with open("log.txt", "a", encoding="utf-8") as f:
    from datetime import datetime
    f.write(f"[{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}] Application started\n")

# ตัวอย่าง: เขียน report
data = [
    ("Alice", 85, "B"),
    ("Bob", 92, "A"),
    ("Charlie", 78, "C"),
]

with open("report.txt", "w", encoding="utf-8") as f:
    f.write("=" * 40 + "\n")
    f.write("Student Report\n")
    f.write("=" * 40 + "\n")
    f.write(f"{'Name':<15} {'Score':>6} {'Grade':>6}\n")
    f.write("-" * 30 + "\n")
    for name, score, grade in data:
        f.write(f"{name:<15} {score:>6} {grade:>6}\n")
    f.write("-" * 30 + "\n")
    avg = sum(s for _, s, _ in data) / len(data)
    f.write(f"{'Average':<15} {avg:>6.1f}\n")

# อ่านผลลัพธ์
with open("report.txt", "r", encoding="utf-8") as f:
    print(f.read())
```

---

## 4. Binary File Mode

```python
# binary mode สำหรับ image, audio, ฯลฯ
# สร้างไฟล์ binary ทดสอบ
data = bytes([0x89, 0x50, 0x4E, 0x47])  # PNG magic bytes
with open("test.bin", "wb") as f:
    f.write(data)
    f.write(b"\x00" * 100)  # padding

# อ่านไฟล์ binary
with open("test.bin", "rb") as f:
    header = f.read(4)
    print(f"Header: {header.hex()}")       # 89504e47
    print(f"Header bytes: {list(header)}")  # [137, 80, 78, 71]

# คัดลอกไฟล์ (ทุก type)
def copy_file(src, dst, chunk_size=8192):
    """คัดลอกไฟล์ทีละ chunk"""
    with open(src, "rb") as fin, open(dst, "wb") as fout:
        while True:
            chunk = fin.read(chunk_size)
            if not chunk:
                break
            fout.write(chunk)
    return True

copy_file("test.bin", "test_copy.bin")
print("คัดลอกสำเร็จ")

# ตรวจสอบว่าไฟล์เหมือนกัน
with open("test.bin", "rb") as f1, open("test_copy.bin", "rb") as f2:
    print(f"ไฟล์เหมือนกัน: {f1.read() == f2.read()}")
```

---

## 5. CSV Files

```python
import csv

# เขียน CSV
students = [
    {"name": "Alice", "age": 22, "grade": 85, "city": "Bangkok"},
    {"name": "Bob", "age": 20, "grade": 92, "city": "Chiang Mai"},
    {"name": "Charlie", "age": 23, "grade": 78, "city": "Phuket"},
    {"name": "Diana", "age": 21, "grade": 95, "city": "Bangkok"},
]

# DictWriter - เขียนจาก dict
with open("students.csv", "w", newline="", encoding="utf-8") as f:
    fieldnames = ["name", "age", "grade", "city"]
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    
    writer.writeheader()          # เขียน header
    writer.writerows(students)    # เขียนทุก row

print("เขียน CSV สำเร็จ")

# อ่าน CSV ด้วย DictReader
with open("students.csv", "r", encoding="utf-8") as f:
    reader = csv.DictReader(f)
    students_data = list(reader)

print(f"\nอ่านได้ {len(students_data)} records")
for s in students_data:
    print(f"  {s['name']:10} อายุ {s['age']} คะแนน {s['grade']}")

# writer/reader พื้นฐาน
with open("numbers.csv", "w", newline="") as f:
    writer = csv.writer(f)
    writer.writerow(["x", "x^2", "x^3"])  # header
    for x in range(1, 11):
        writer.writerow([x, x**2, x**3])

with open("numbers.csv", "r") as f:
    reader = csv.reader(f)
    header = next(reader)  # อ่าน header
    print(f"\nHeader: {header}")
    for row in reader:
        print(f"  {row}")

# CSV กับ options
# quotechar, delimiter, quoting
with open("data.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.writer(f, delimiter="\t", quotechar='"')
    writer.writerows([
        ["Name", "Message"],
        ["Alice", "Hello, World!"],  # มี comma - ต้อง quote
        ["Bob", 'He said "hi"'],     # มี quote - ต้อง escape
    ])

# ตัวอย่างจริง: วิเคราะห์ CSV
def analyze_csv(filepath):
    with open(filepath, "r", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        rows = list(reader)
    
    if not rows:
        return {}
    
    # คำนวณสถิติสำหรับ numeric columns
    stats = {}
    for col in rows[0].keys():
        values = []
        for row in rows:
            try:
                values.append(float(row[col]))
            except ValueError:
                pass
        
        if values:
            stats[col] = {
                "count": len(values),
                "sum": sum(values),
                "avg": sum(values) / len(values),
                "min": min(values),
                "max": max(values),
            }
    
    return stats

stats = analyze_csv("students.csv")
print("\n=== CSV Statistics ===")
for col, s in stats.items():
    print(f"\n{col}:")
    for k, v in s.items():
        print(f"  {k}: {v:.2f}")
```

---

## 6. JSON Files

```python
import json

# Python objects ที่แปลงเป็น JSON ได้
data = {
    "name": "Alice",
    "age": 30,
    "active": True,
    "score": 85.5,
    "tags": ["python", "django"],
    "address": {
        "city": "Bangkok",
        "country": "Thailand"
    },
    "nothing": None
}

# json.dumps() - แปลง Python object เป็น JSON string
json_str = json.dumps(data)
print(json_str[:80])

# json.dumps() กับ options
json_pretty = json.dumps(data, indent=2, ensure_ascii=False, sort_keys=True)
print(json_pretty)

# json.loads() - แปลง JSON string เป็น Python object
parsed = json.loads(json_str)
print(type(parsed))      # <class 'dict'>
print(parsed["name"])    # Alice
print(parsed["tags"])    # ['python', 'django']

# เขียน JSON ไฟล์
with open("data.json", "w", encoding="utf-8") as f:
    json.dump(data, f, indent=2, ensure_ascii=False)
print("เขียน JSON สำเร็จ")

# อ่าน JSON ไฟล์
with open("data.json", "r", encoding="utf-8") as f:
    loaded = json.load(f)
print(f"อ่านได้: {loaded['name']}, {loaded['age']}")

# Custom JSON encoder
from datetime import datetime, date
import decimal

class CustomEncoder(json.JSONEncoder):
    """Encoder ที่รองรับ datetime และ Decimal"""
    def default(self, obj):
        if isinstance(obj, (datetime, date)):
            return obj.isoformat()
        if isinstance(obj, decimal.Decimal):
            return float(obj)
        return super().default(obj)

data2 = {
    "name": "Alice",
    "created_at": datetime(2024, 1, 15, 10, 30),
    "price": decimal.Decimal("99.99"),
}

json_str = json.dumps(data2, cls=CustomEncoder, indent=2)
print(json_str)

# JSON Lines format (jsonl) - 1 JSON object ต่อ 1 บรรทัด (สำหรับ big data)
records = [
    {"id": 1, "event": "login", "user": "alice"},
    {"id": 2, "event": "view", "user": "bob"},
    {"id": 3, "event": "purchase", "user": "alice"},
]

with open("events.jsonl", "w", encoding="utf-8") as f:
    for record in records:
        f.write(json.dumps(record) + "\n")

# อ่าน jsonl
with open("events.jsonl", "r", encoding="utf-8") as f:
    events = [json.loads(line) for line in f]
print(events)
```

---

## 7. pathlib - Modern Path Handling

```python
from pathlib import Path
import os

# สร้าง Path object
p = Path(".")                     # current directory
home = Path.home()                # home directory
cwd = Path.cwd()                  # current working directory
docs = Path("/home/user/documents")

print(f"CWD: {cwd}")
print(f"Home: {home}")

# Path operations
path = Path("data/files/report.csv")

print(path.name)        # report.csv
print(path.stem)        # report (ไม่มี extension)
print(path.suffix)      # .csv
print(path.suffixes)    # ['.csv']
print(path.parent)      # data/files
print(path.parents[0])  # data/files
print(path.parents[1])  # data
print(path.parts)       # ('data', 'files', 'report.csv')

# Path ด้วย / operator
base = Path("/home/user")
full_path = base / "documents" / "report.txt"
print(full_path)  # /home/user/documents/report.txt

# ตรวจสอบ
p = Path("test.txt")
print(p.exists())     # True/False
print(p.is_file())    # True/False
print(p.is_dir())     # True/False

# สร้าง directory
new_dir = Path("test_dir/subdir")
new_dir.mkdir(parents=True, exist_ok=True)  # parents=True สร้าง parent ด้วย
print(f"สร้าง: {new_dir}")

# Glob - ค้นหาไฟล์
cwd = Path(".")
txt_files = list(cwd.glob("*.txt"))
print(f"TXT files: {txt_files}")

all_py = list(cwd.rglob("*.py"))  # recursive glob
print(f"Python files: {all_py}")

# อ่าน/เขียนด้วย pathlib
p = Path("hello.txt")
p.write_text("Hello, World!\nสวัสดีโลก\n", encoding="utf-8")
content = p.read_text(encoding="utf-8")
print(content)

# Binary
p_bin = Path("data.bin")
p_bin.write_bytes(b"\x89PNG")
data = p_bin.read_bytes()
print(data.hex())

# File stats
p = Path("test.txt")
if p.exists():
    stat = p.stat()
    print(f"ขนาด: {stat.st_size} bytes")
    from datetime import datetime
    modified = datetime.fromtimestamp(stat.st_mtime)
    print(f"แก้ไขล่าสุด: {modified}")

# rename / replace
# p.rename("new_name.txt")
# p.replace("destination.txt")  # overwrite

# ลบไฟล์/directory
p_del = Path("hello.txt")
if p_del.exists():
    p_del.unlink()  # ลบไฟล์

# ลบ directory ว่าง
empty_dir = Path("test_dir/subdir")
if empty_dir.exists():
    empty_dir.rmdir()

# ลบ directory และทุกอย่างข้างใน
import shutil
if Path("test_dir").exists():
    shutil.rmtree("test_dir")
```

---

## 8. File/Directory Operations

```python
import shutil
import os
from pathlib import Path

# สร้าง directory structure สำหรับทดสอบ
base = Path("test_project")
(base / "src").mkdir(parents=True, exist_ok=True)
(base / "tests").mkdir(exist_ok=True)
(base / "docs").mkdir(exist_ok=True)

# สร้างไฟล์ทดสอบ
(base / "src" / "main.py").write_text("# main.py\nprint('Hello')")
(base / "src" / "utils.py").write_text("# utils.py")
(base / "tests" / "test_main.py").write_text("# tests")
(base / "README.md").write_text("# Test Project")

# แสดง directory tree
def show_tree(path: Path, indent: int = 0):
    """แสดง directory tree"""
    print("  " * indent + path.name + ("/" if path.is_dir() else ""))
    if path.is_dir():
        for child in sorted(path.iterdir()):
            show_tree(child, indent + 1)

print("=== Project Structure ===")
show_tree(base)

# คัดลอก directory
shutil.copytree(base, Path("test_project_backup"), dirs_exist_ok=True)
print("\nคัดลอก directory สำเร็จ")

# zip directory
shutil.make_archive("test_project_archive", "zip", base)
print("สร้าง zip สำเร็จ")

# unzip
shutil.unpack_archive("test_project_archive.zip", "test_project_unzipped")
print("แตกไฟล์สำเร็จ")

# disk usage
def get_dir_size(path: Path) -> int:
    """คำนวณขนาด directory"""
    return sum(f.stat().st_size for f in path.rglob("*") if f.is_file())

size = get_dir_size(base)
print(f"\nขนาด test_project: {size} bytes")

# ค้นหาไฟล์ตาม pattern
print("\nไฟล์ Python ใน test_project:")
for f in sorted(base.rglob("*.py")):
    print(f"  {f.relative_to(base)}")

# Cleanup
import os
for f in ["test_project_archive.zip"]:
    if os.path.exists(f):
        os.remove(f)
for d in ["test_project", "test_project_backup", "test_project_unzipped"]:
    if Path(d).exists():
        shutil.rmtree(d)
```

---

## 9. ตัวอย่างโปรแกรมจริง: Data Logger

```python
"""
ระบบ Data Logger สำหรับบันทึกข้อมูล sensor
"""
import json
import csv
from pathlib import Path
from datetime import datetime
from typing import Dict, List, Any, Optional
import os

class DataLogger:
    """บันทึกข้อมูลลงไฟล์ต่างๆ"""
    
    def __init__(self, base_dir: str = "logs"):
        self.base_dir = Path(base_dir)
        self.base_dir.mkdir(parents=True, exist_ok=True)
        self._session_id = datetime.now().strftime("%Y%m%d_%H%M%S")
    
    @property
    def log_dir(self) -> Path:
        """Directory ของ session นี้"""
        d = self.base_dir / self._session_id
        d.mkdir(exist_ok=True)
        return d
    
    def log_text(self, message: str, level: str = "INFO") -> None:
        """บันทึก log ธรรมดา"""
        log_file = self.log_dir / "app.log"
        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S.%f")[:-3]
        
        with open(log_file, "a", encoding="utf-8") as f:
            f.write(f"[{timestamp}] [{level:5}] {message}\n")
    
    def log_csv(self, data: Dict[str, Any], filename: str = "data.csv") -> None:
        """บันทึกข้อมูลเป็น CSV"""
        csv_file = self.log_dir / filename
        file_exists = csv_file.exists()
        
        with open(csv_file, "a", newline="", encoding="utf-8") as f:
            writer = csv.DictWriter(f, fieldnames=list(data.keys()))
            if not file_exists:
                writer.writeheader()
            writer.writerow(data)
    
    def log_json(self, data: Any, filename: str = None) -> Path:
        """บันทึกข้อมูลเป็น JSON"""
        if filename is None:
            filename = f"data_{datetime.now().strftime('%H%M%S')}.json"
        
        json_file = self.log_dir / filename
        with open(json_file, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=2, default=str)
        
        return json_file
    
    def read_csv_log(self, filename: str = "data.csv") -> List[Dict]:
        """อ่านข้อมูลจาก CSV log"""
        csv_file = self.log_dir / filename
        if not csv_file.exists():
            return []
        
        with open(csv_file, "r", encoding="utf-8") as f:
            reader = csv.DictReader(f)
            return list(reader)
    
    def get_stats(self, filename: str = "data.csv", column: str = None) -> Dict:
        """คำนวณสถิติจาก CSV"""
        data = self.read_csv_log(filename)
        if not data or not column:
            return {"count": len(data)}
        
        values = []
        for row in data:
            try:
                values.append(float(row[column]))
            except (ValueError, KeyError):
                pass
        
        if not values:
            return {"count": len(data)}
        
        return {
            "count": len(values),
            "sum": sum(values),
            "avg": sum(values) / len(values),
            "min": min(values),
            "max": max(values),
        }
    
    def rotate_logs(self, max_sessions: int = 10) -> int:
        """ลบ session เก่าถ้ามีมากเกิน max_sessions"""
        sessions = sorted(
            [d for d in self.base_dir.iterdir() if d.is_dir()],
            key=lambda d: d.name
        )
        
        deleted = 0
        while len(sessions) > max_sessions:
            old_session = sessions.pop(0)
            import shutil
            shutil.rmtree(old_session)
            deleted += 1
        
        return deleted
    
    def export_summary(self) -> Path:
        """สร้าง summary report"""
        summary_file = self.log_dir / "summary.json"
        
        csv_files = list(self.log_dir.glob("*.csv"))
        log_files = list(self.log_dir.glob("*.log"))
        
        summary = {
            "session_id": self._session_id,
            "generated_at": datetime.now().isoformat(),
            "files": {
                "csv": [f.name for f in csv_files],
                "log": [f.name for f in log_files],
            },
        }
        
        # รวมสถิติจากทุก CSV
        all_stats = {}
        for csv_file in csv_files:
            data = []
            with open(csv_file, "r", encoding="utf-8") as f:
                data = list(csv.DictReader(f))
            if data:
                all_stats[csv_file.name] = {"rows": len(data)}
        
        summary["data_summary"] = all_stats
        
        with open(summary_file, "w", encoding="utf-8") as f:
            json.dump(summary, f, ensure_ascii=False, indent=2)
        
        return summary_file

# ทดสอบ
import random
import time

logger = DataLogger("demo_logs")

logger.log_text("Logger started", "INFO")
logger.log_text("Sensor initialized", "INFO")

# จำลองการบันทึก sensor data
print("=== กำลังบันทึก Sensor Data ===")
for i in range(5):
    sensor_data = {
        "timestamp": datetime.now().isoformat(),
        "temperature": round(25 + random.uniform(-5, 5), 2),
        "humidity": round(60 + random.uniform(-10, 10), 2),
        "pressure": round(1013 + random.uniform(-20, 20), 2),
    }
    logger.log_csv(sensor_data, "sensors.csv")
    logger.log_text(f"Reading {i+1}: temp={sensor_data['temperature']}°C", "DEBUG")
    print(f"  บันทึก: {sensor_data['temperature']}°C, {sensor_data['humidity']}%")

logger.log_text("All readings complete", "INFO")

# แสดงสถิติ
stats = logger.get_stats("sensors.csv", "temperature")
print(f"\n=== Temperature Stats ===")
for key, val in stats.items():
    print(f"  {key}: {val:.2f}" if isinstance(val, float) else f"  {key}: {val}")

# Export summary
summary_path = logger.export_summary()
print(f"\nSummary: {summary_path}")

# อ่าน summary
with open(summary_path, "r", encoding="utf-8") as f:
    summary = json.load(f)
print(f"Session: {summary['session_id']}")

# Cleanup
import shutil
if Path("demo_logs").exists():
    shutil.rmtree("demo_logs")
```

---

## 10. Exercises

### Exercise 1: Config File Reader

```python
"""
สร้าง Config File Reader ที่รองรับ:
1. อ่าน/เขียน INI-like format
2. Section support: [section_name]
3. Comments: # หรือ ;
4. Key-value pairs: key = value
"""
from pathlib import Path
from typing import Dict, Optional
import re

class ConfigParser:
    def __init__(self):
        self._data: Dict[str, Dict[str, str]] = {"DEFAULT": {}}
        self._current_section = "DEFAULT"
    
    def read(self, filepath: str) -> "ConfigParser":
        with open(filepath, "r", encoding="utf-8") as f:
            for line in f:
                line = line.strip()
                if not line or line.startswith(("#", ";")):
                    continue
                
                section_match = re.match(r'^\[(.+)\]$', line)
                if section_match:
                    self._current_section = section_match.group(1)
                    self._data.setdefault(self._current_section, {})
                    continue
                
                kv_match = re.match(r'^(\w+)\s*=\s*(.*)$', line)
                if kv_match:
                    key, value = kv_match.group(1), kv_match.group(2).strip()
                    if value.startswith('"') and value.endswith('"'):
                        value = value[1:-1]
                    self._data[self._current_section][key] = value
        
        return self
    
    def get(self, section: str, key: str, fallback: str = None) -> Optional[str]:
        return self._data.get(section, {}).get(key, fallback)
    
    def sections(self):
        return [s for s in self._data if s != "DEFAULT"]
    
    def write(self, filepath: str) -> None:
        with open(filepath, "w", encoding="utf-8") as f:
            for section, items in self._data.items():
                if items:
                    f.write(f"[{section}]\n")
                    for key, value in items.items():
                        f.write(f"{key} = {value}\n")
                    f.write("\n")

# สร้างไฟล์ config ทดสอบ
config_content = """
# Application Configuration
[database]
host = localhost
port = 5432
name = myapp
user = admin
password = secret123

[cache]
backend = redis
host = localhost
port = 6379
ttl = 300

[app]
debug = false
secret_key = "my-secret-key-here"
allowed_hosts = localhost,127.0.0.1
"""

with open("config.ini", "w", encoding="utf-8") as f:
    f.write(config_content)

# ทดสอบ
cfg = ConfigParser()
cfg.read("config.ini")

print("Sections:", cfg.sections())
print(f"DB Host: {cfg.get('database', 'host')}")
print(f"DB Port: {cfg.get('database', 'port')}")
print(f"Cache TTL: {cfg.get('cache', 'ttl')}")
print(f"Missing key: {cfg.get('app', 'missing', 'DEFAULT_VALUE')}")

# Cleanup
import os
os.remove("config.ini")
```

### Exercise 2: File Backup System

```python
"""
ระบบ Backup ไฟล์:
1. backup ไฟล์และ directory
2. เก็บ history (timestamp)
3. restore จาก backup
4. cleanup backup เก่า
"""
import shutil
import json
from pathlib import Path
from datetime import datetime
from typing import List, Optional

class BackupSystem:
    def __init__(self, backup_dir: str = "backups"):
        self.backup_dir = Path(backup_dir)
        self.backup_dir.mkdir(parents=True, exist_ok=True)
        self.manifest_file = self.backup_dir / "manifest.json"
        self._load_manifest()
    
    def _load_manifest(self):
        if self.manifest_file.exists():
            with open(self.manifest_file, "r") as f:
                self.manifest = json.load(f)
        else:
            self.manifest = {"backups": []}
    
    def _save_manifest(self):
        with open(self.manifest_file, "w") as f:
            json.dump(self.manifest, f, indent=2, default=str)
    
    def backup(self, source: str, tag: str = "") -> str:
        """สร้าง backup"""
        source_path = Path(source)
        if not source_path.exists():
            raise FileNotFoundError(f"ไม่พบ: {source}")
        
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        backup_name = f"{source_path.name}_{timestamp}"
        backup_path = self.backup_dir / backup_name
        
        if source_path.is_dir():
            shutil.copytree(source_path, backup_path)
        else:
            shutil.copy2(source_path, backup_path)
        
        record = {
            "id": timestamp,
            "source": str(source_path),
            "backup": str(backup_path),
            "type": "dir" if source_path.is_dir() else "file",
            "tag": tag,
            "created": datetime.now().isoformat(),
            "size": self._get_size(backup_path),
        }
        self.manifest["backups"].append(record)
        self._save_manifest()
        
        return timestamp
    
    def restore(self, backup_id: str, destination: Optional[str] = None) -> bool:
        """restore จาก backup"""
        record = next(
            (b for b in self.manifest["backups"] if b["id"] == backup_id), None
        )
        if not record:
            raise ValueError(f"ไม่พบ backup {backup_id}")
        
        backup_path = Path(record["backup"])
        dest = Path(destination) if destination else Path(record["source"])
        
        if backup_path.is_dir():
            if dest.exists():
                shutil.rmtree(dest)
            shutil.copytree(backup_path, dest)
        else:
            shutil.copy2(backup_path, dest)
        
        return True
    
    def list_backups(self) -> List[dict]:
        return self.manifest["backups"]
    
    def cleanup(self, keep_latest: int = 3) -> int:
        """ลบ backup เก่า"""
        by_source = {}
        for b in self.manifest["backups"]:
            by_source.setdefault(b["source"], []).append(b)
        
        deleted = 0
        for source, backups in by_source.items():
            sorted_backups = sorted(backups, key=lambda x: x["created"])
            to_delete = sorted_backups[:-keep_latest]
            for b in to_delete:
                p = Path(b["backup"])
                if p.is_dir():
                    shutil.rmtree(p)
                elif p.exists():
                    p.unlink()
                self.manifest["backups"].remove(b)
                deleted += 1
        
        self._save_manifest()
        return deleted
    
    def _get_size(self, path: Path) -> int:
        if path.is_file():
            return path.stat().st_size
        return sum(f.stat().st_size for f in path.rglob("*") if f.is_file())

# ทดสอบ
# สร้างไฟล์ทดสอบ
test_dir = Path("test_data")
test_dir.mkdir(exist_ok=True)
(test_dir / "file1.txt").write_text("version 1")
(test_dir / "file2.txt").write_text("version 1")

bs = BackupSystem("test_backups")

# สร้าง backups
id1 = bs.backup("test_data", tag="initial")
print(f"Backup 1: {id1}")

# แก้ไขไฟล์
(test_dir / "file1.txt").write_text("version 2")
id2 = bs.backup("test_data", tag="after_edit")
print(f"Backup 2: {id2}")

# แสดง backups
print("\n=== Backup History ===")
for b in bs.list_backups():
    print(f"  [{b['id']}] {b['tag']} ({b['size']} bytes)")

# Cleanup
import shutil
if test_dir.exists():
    shutil.rmtree(test_dir)
if Path("test_backups").exists():
    shutil.rmtree("test_backups")
```

---

## 11. สรุป Part 012

### สิ่งที่เรียนรู้:

✅ **open()** - modes r/w/a/x/b/t  
✅ **with statement** - context manager, ปิดอัตโนมัติ  
✅ **read/readline/readlines** - การอ่านไฟล์  
✅ **write/writelines** - การเขียนไฟล์  
✅ **seek/tell** - การย้าย file pointer  
✅ **Binary mode** - อ่าน/เขียนไฟล์ binary  
✅ **csv module** - DictReader, DictWriter  
✅ **json module** - loads/dumps/load/dump  
✅ **pathlib** - Path operations ที่ modern  
✅ **shutil** - copy, move, archive operations  

### Quick Reference:

```python
# Read
with open("file.txt", "r", encoding="utf-8") as f:
    content = f.read()           # ทั้งหมด
    lines = f.readlines()        # list of lines
    for line in f: ...           # iterate

# Write
with open("file.txt", "w", encoding="utf-8") as f:
    f.write("text\n")
    f.writelines(["a\n", "b\n"])

# Append
with open("file.txt", "a", encoding="utf-8") as f:
    f.write("more text\n")

# CSV
import csv
with open("data.csv", "r") as f:
    reader = csv.DictReader(f)
    rows = list(reader)

# JSON
import json
data = json.load(open("file.json"))
json.dump(data, open("file.json", "w"), indent=2)

# pathlib
from pathlib import Path
p = Path("dir") / "file.txt"
p.read_text(), p.write_text("...")
p.exists(), p.is_file(), p.is_dir()
list(p.parent.glob("*.txt"))
```

---

## ➡️ ถัดไป: Part 013 - Exception Handling

ใน Part ถัดไป เราจะเรียนรู้:
- try/except/else/finally
- Custom exceptions
- Exception hierarchy
- Context managers
- Best practices

---

*Part 012/100+ | Python Course - Beginner to World-Class*
