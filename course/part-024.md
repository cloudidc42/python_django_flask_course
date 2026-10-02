# Part 024 - JSON and CSV (การจัดการข้อมูล JSON และ CSV)

## เป้าหมาย
- ใช้ `json` module: loads/dumps/load/dump
- ใช้ `csv` module: reader/writer/DictReader/DictWriter
- แนะนำ pandas สำหรับ CSV
- ตัวอย่าง data processing จริง

---

## 1. JSON Module

### JSON Basics

```python
import json

# Python types ที่แปลงเป็น JSON ได้
data = {
    "name": "Alice",          # str -> string
    "age": 25,                # int -> number
    "salary": 75000.50,       # float -> number
    "is_active": True,        # bool -> true/false
    "nickname": None,         # None -> null
    "tags": ["python", "dev"], # list -> array
    "address": {              # dict -> object
        "street": "123 Main St",
        "city": "Bangkok",
        "country": "Thailand"
    }
}

# dumps() - แปลง Python -> JSON string
json_str = json.dumps(data)
print(type(json_str))  # <class 'str'>
print(json_str[:50])   # {"name": "Alice", "age": 25, "salary": 75000.5

# dumps กับ options
formatted = json.dumps(data, indent=4, ensure_ascii=False, sort_keys=True)
print(formatted)

# loads() - แปลง JSON string -> Python
parsed = json.loads(json_str)
print(type(parsed))        # <class 'dict'>
print(parsed["name"])      # Alice
print(type(parsed["age"])) # <class 'int'>

# ข้อมูลภาษาไทย
thai_data = {"ชื่อ": "สมชาย", "อายุ": 30, "เมือง": "กรุงเทพ"}

# ensure_ascii=False เพื่อรักษา unicode characters
thai_json = json.dumps(thai_data, ensure_ascii=False, indent=2)
print(thai_json)
# {"ชื่อ": "สมชาย", "อายุ": 30, "เมือง": "กรุงเทพ"}

# ถ้า ensure_ascii=True (default)
ascii_json = json.dumps(thai_data)
print(ascii_json)  
# {"\\u0e0a\\u0e37\\u0e48\\u0e2d": ...} <- escaped unicode
```

### JSON File Operations

```python
import json
import os

# dump() - เขียน JSON ลงไฟล์
data = {
    "users": [
        {"id": 1, "name": "Alice", "email": "alice@example.com"},
        {"id": 2, "name": "Bob", "email": "bob@example.com"},
    ],
    "total": 2
}

# เขียนไฟล์
with open("/tmp/users.json", "w", encoding="utf-8") as f:
    json.dump(data, f, ensure_ascii=False, indent=2)

# load() - อ่านจากไฟล์
with open("/tmp/users.json", "r", encoding="utf-8") as f:
    loaded = json.load(f)

print(f"Loaded {loaded['total']} users")
for user in loaded["users"]:
    print(f"  {user['id']}: {user['name']} ({user['email']})")

# ล้างไฟล์
os.unlink("/tmp/users.json")
```

### Custom JSON Encoder/Decoder

```python
import json
from datetime import datetime, date
from decimal import Decimal
from typing import Any

class CustomEncoder(json.JSONEncoder):
    """Custom encoder สำหรับ types ที่ JSON ไม่รองรับ"""
    
    def default(self, obj: Any) -> Any:
        if isinstance(obj, datetime):
            return {"__type__": "datetime", "value": obj.isoformat()}
        elif isinstance(obj, date):
            return {"__type__": "date", "value": obj.isoformat()}
        elif isinstance(obj, Decimal):
            return {"__type__": "decimal", "value": str(obj)}
        elif isinstance(obj, set):
            return {"__type__": "set", "value": list(obj)}
        elif isinstance(obj, bytes):
            import base64
            return {"__type__": "bytes", "value": base64.b64encode(obj).decode()}
        
        return super().default(obj)


class CustomDecoder(json.JSONDecoder):
    """Custom decoder"""
    
    def __init__(self, *args, **kwargs):
        super().__init__(object_hook=self.object_hook, *args, **kwargs)
    
    def object_hook(self, obj: dict) -> Any:
        if "__type__" in obj:
            type_name = obj["__type__"]
            if type_name == "datetime":
                return datetime.fromisoformat(obj["value"])
            elif type_name == "date":
                return date.fromisoformat(obj["value"])
            elif type_name == "decimal":
                return Decimal(obj["value"])
            elif type_name == "set":
                return set(obj["value"])
        return obj


# ทดสอบ
data = {
    "created_at": datetime(2024, 10, 15, 14, 30, 0),
    "birthday": date(1990, 5, 15),
    "price": Decimal("99.99"),
    "tags": {"python", "web", "api"},
    "data": b"binary data"
}

# Encode
json_str = json.dumps(data, cls=CustomEncoder, indent=2, ensure_ascii=False)
print(json_str)

# Decode
restored = json.loads(json_str, cls=CustomDecoder)
print(f"\nRestored datetime: {restored['created_at']}")
print(f"Restored date: {restored['birthday']}")
print(f"Restored decimal: {restored['price']}")
print(f"Restored set: {restored['tags']}")

# Alternative: JSONDecodable Protocol
class JSONSerializable:
    """Base class สำหรับ classes ที่ serialize เป็น JSON ได้"""
    
    def to_json(self) -> dict:
        raise NotImplementedError
    
    @classmethod
    def from_json(cls, data: dict):
        raise NotImplementedError
    
    def __repr__(self):
        return f"{self.__class__.__name__}({self.to_json()})"


from dataclasses import dataclass, asdict

@dataclass
class Product(JSONSerializable):
    product_id: int
    name: str
    price: float
    created_at: datetime = None
    
    def __post_init__(self):
        if self.created_at is None:
            self.created_at = datetime.now()
    
    def to_json(self) -> dict:
        d = asdict(self)
        d["created_at"] = self.created_at.isoformat()
        return d
    
    @classmethod
    def from_json(cls, data: dict):
        data = data.copy()
        if "created_at" in data:
            data["created_at"] = datetime.fromisoformat(data["created_at"])
        return cls(**data)


p = Product(1, "Python Book", 299.0)
json_data = json.dumps(p.to_json(), ensure_ascii=False)
print(json_data)

restored_p = Product.from_json(json.loads(json_data))
print(restored_p)
```

---

## 2. CSV Module

### CSV Basics

```python
import csv
import io

# === Writer ===
# สร้าง CSV ใน memory
output = io.StringIO()

writer = csv.writer(output)
writer.writerow(["Name", "Age", "Email", "Salary"])  # header
writer.writerow(["Alice", 25, "alice@example.com", 75000])
writer.writerow(["Bob", 30, "bob@example.com", 85000])
writer.writerow(["Charlie", 28, "charlie@example.com", 70000])

csv_content = output.getvalue()
print(csv_content)

# === Reader ===
reader = csv.reader(io.StringIO(csv_content))
for row in reader:
    print(row)  # list of strings

# ข้ามบรรทัดแรก (header)
reader = csv.reader(io.StringIO(csv_content))
header = next(reader)
print(f"Columns: {header}")

for row in reader:
    name, age, email, salary = row
    print(f"{name} (age {age}): {email} - {float(salary):,.0f}")

# === DictWriter ===
output = io.StringIO()

fieldnames = ["name", "age", "email", "salary"]
writer = csv.DictWriter(output, fieldnames=fieldnames)

writer.writeheader()  # เขียน header อัตโนมัติ

employees = [
    {"name": "สมชาย", "age": 28, "email": "somchai@example.com", "salary": 55000},
    {"name": "สมหญิง", "age": 26, "email": "somying@example.com", "salary": 50000},
]

for emp in employees:
    writer.writerow(emp)

# writerows - เขียนหลายแถวพร้อมกัน
more_employees = [
    {"name": "มานี", "age": 32, "email": "manee@example.com", "salary": 65000},
    {"name": "วิชัย", "age": 35, "email": "wichai@example.com", "salary": 70000},
]
writer.writerows(more_employees)

csv_content = output.getvalue()
print(csv_content)

# === DictReader ===
reader = csv.DictReader(io.StringIO(csv_content))
for row in reader:
    # row เป็น dict
    print(f"{row['name']}: {row['email']}")
```

### CSV Dialects และ Options

```python
import csv
import io

# Dialect: กำหนด format ของ CSV
# Default: comma separator, double-quote quoting

# Tab-separated (TSV)
tsv_data = "Name\tAge\tCity\nAlice\t25\tBangkok\nBob\t30\tChiangMai"
reader = csv.reader(io.StringIO(tsv_data), delimiter='\t')
for row in reader:
    print(row)

# Semicolon separator (ใช้ใน Excel ยุโรป)
semi_data = "Name;Age;Score\nAlice;25;85.5\nBob;30;92.0"
reader = csv.DictReader(io.StringIO(semi_data), delimiter=';')
for row in reader:
    print(dict(row))

# Custom quoting
data_with_commas = [
    ["Name", "Address", "Score"],
    ["Alice", "123 Main St, Bangkok", 85],
    ["Bob", 'He said "Hello"', 90],
]

output = io.StringIO()
writer = csv.writer(output, 
                    quoting=csv.QUOTE_NONNUMERIC,  # quote strings
                    quotechar='"')
for row in data_with_commas:
    writer.writerow(row)

print(output.getvalue())

# Register custom dialect
csv.register_dialect('thai', 
                     delimiter=',',
                     quoting=csv.QUOTE_MINIMAL,
                     lineterminator='\r\n')

# ใช้ custom dialect
output = io.StringIO()
writer = csv.writer(output, dialect='thai')
writer.writerow(["ชื่อ", "อายุ", "แผนก"])
writer.writerow(["สมชาย", 28, "วิศวกรรม"])
print(output.getvalue())
```

### CSV File Processing

```python
import csv
import os
from typing import List, Dict, Any, Optional, Iterator
from collections import defaultdict

class CSVProcessor:
    """ประมวลผล CSV files"""
    
    def __init__(self, filepath: str, encoding: str = 'utf-8-sig'):
        self.filepath = filepath
        self.encoding = encoding
    
    def read_all(self) -> List[Dict]:
        """อ่านทั้งไฟล์"""
        with open(self.filepath, 'r', encoding=self.encoding, newline='') as f:
            reader = csv.DictReader(f)
            return list(reader)
    
    def read_lazy(self) -> Iterator[Dict]:
        """อ่านแบบ lazy"""
        with open(self.filepath, 'r', encoding=self.encoding, newline='') as f:
            reader = csv.DictReader(f)
            yield from reader
    
    def write(self, data: List[Dict], fieldnames: List[str] = None):
        """เขียน CSV"""
        if not data:
            return
        
        if fieldnames is None:
            fieldnames = list(data[0].keys())
        
        with open(self.filepath, 'w', encoding=self.encoding, 
                  newline='') as f:
            writer = csv.DictWriter(f, fieldnames=fieldnames)
            writer.writeheader()
            writer.writerows(data)
    
    def append(self, rows: List[Dict]):
        """เพิ่มแถวใหม่"""
        with open(self.filepath, 'a', encoding=self.encoding, 
                  newline='') as f:
            if rows:
                writer = csv.DictWriter(f, fieldnames=list(rows[0].keys()))
                writer.writerows(rows)
    
    def filter_rows(self, predicate) -> List[Dict]:
        """กรองแถวตาม predicate"""
        return [row for row in self.read_lazy() if predicate(row)]
    
    def transform(self, func) -> List[Dict]:
        """แปลงข้อมูลทุกแถว"""
        return [func(row) for row in self.read_lazy()]
    
    def aggregate(self, group_by: str, agg_col: str, 
                  func=None) -> Dict[str, float]:
        """สรุปข้อมูลตาม group"""
        groups = defaultdict(list)
        
        for row in self.read_lazy():
            key = row[group_by]
            try:
                value = float(row[agg_col])
                groups[key].append(value)
            except (ValueError, KeyError):
                pass
        
        if func is None:
            func = lambda vals: sum(vals) / len(vals)  # default: mean
        
        return {key: func(values) for key, values in groups.items()}
    
    def get_column(self, column: str) -> List[str]:
        """ดึงข้อมูล column เดียว"""
        return [row[column] for row in self.read_lazy() if column in row]
    
    def count_by(self, column: str) -> Dict[str, int]:
        """นับจำนวนตาม column value"""
        counts = defaultdict(int)
        for row in self.read_lazy():
            if column in row:
                counts[row[column]] += 1
        return dict(counts)
    
    def to_json(self, filepath: str):
        """แปลง CSV เป็น JSON"""
        import json
        data = self.read_all()
        with open(filepath, 'w', encoding='utf-8') as f:
            json.dump(data, f, ensure_ascii=False, indent=2)


# ทดสอบ CSVProcessor
import tempfile

# สร้างข้อมูลทดสอบ
test_data = [
    {"name": "Alice", "dept": "Engineering", "salary": "75000", "age": "28"},
    {"name": "Bob", "dept": "Marketing", "salary": "55000", "age": "35"},
    {"name": "Charlie", "dept": "Engineering", "salary": "85000", "age": "32"},
    {"name": "Dave", "dept": "HR", "salary": "50000", "age": "27"},
    {"name": "Eve", "dept": "Engineering", "salary": "90000", "age": "30"},
    {"name": "Frank", "dept": "Marketing", "salary": "60000", "age": "40"},
]

# เขียนไฟล์
with tempfile.NamedTemporaryFile(mode='w', suffix='.csv', delete=False, encoding='utf-8') as f:
    temp_path = f.name

proc = CSVProcessor(temp_path)
proc.write(test_data)

# อ่านและประมวลผล
print("=== Engineering employees ===")
engineers = proc.filter_rows(lambda r: r["dept"] == "Engineering")
for emp in engineers:
    print(f"  {emp['name']}: {float(emp['salary']):,.0f}")

print("\n=== Average salary by dept ===")
avg_salaries = proc.aggregate("dept", "salary")
for dept, avg in sorted(avg_salaries.items()):
    print(f"  {dept}: {avg:,.0f}")

print("\n=== Employee count by dept ===")
counts = proc.count_by("dept")
for dept, count in sorted(counts.items()):
    print(f"  {dept}: {count}")

# ล้างไฟล์
os.unlink(temp_path)
```

---

## 3. pandas สำหรับ CSV

```python
try:
    import pandas as pd
    import numpy as np
    HAS_PANDAS = True
except ImportError:
    print("pandas not installed. Install with: pip install pandas")
    HAS_PANDAS = False

if HAS_PANDAS:
    import io
    
    # สร้างข้อมูลตัวอย่าง
    csv_data = """name,dept,salary,age,hire_date,is_active
Alice,Engineering,75000,28,2022-01-15,True
Bob,Marketing,55000,35,2021-06-01,True
Charlie,Engineering,85000,32,2020-03-10,True
Dave,HR,50000,27,2023-08-20,True
Eve,Engineering,90000,30,2019-11-05,True
Frank,Marketing,60000,40,2018-04-15,False
Grace,Engineering,70000,25,2023-02-28,True
Henry,HR,52000,33,2021-09-15,True
"""
    
    # อ่าน CSV
    df = pd.read_csv(io.StringIO(csv_data), parse_dates=['hire_date'])
    
    print("=== DataFrame ===")
    print(df.head())
    print(f"\nShape: {df.shape}")
    print(f"\nDtypes:\n{df.dtypes}")
    
    # Basic Statistics
    print("\n=== Statistics ===")
    print(df[['salary', 'age']].describe())
    
    # Filtering
    print("\n=== Active Engineers ===")
    engineers = df[(df['dept'] == 'Engineering') & (df['is_active'] == True)]
    print(engineers[['name', 'salary', 'age']])
    
    # Groupby
    print("\n=== Summary by Dept ===")
    summary = df.groupby('dept').agg({
        'name': 'count',
        'salary': ['mean', 'min', 'max'],
        'age': 'mean'
    }).round(2)
    print(summary)
    
    # Apply function
    df['salary_level'] = df['salary'].apply(
        lambda x: 'High' if x >= 75000 else ('Mid' if x >= 60000 else 'Low')
    )
    
    # Sorting
    print("\n=== Top 3 Salary ===")
    print(df.nlargest(3, 'salary')[['name', 'dept', 'salary']])
    
    # String operations
    df['name_upper'] = df['name'].str.upper()
    df['email'] = df['name'].str.lower() + '@company.com'
    
    # Date operations
    from datetime import date
    today = pd.Timestamp.now()
    df['tenure_days'] = (today - df['hire_date']).dt.days
    df['tenure_years'] = (df['tenure_days'] / 365).round(1)
    
    print("\n=== Tenure ===")
    print(df[['name', 'hire_date', 'tenure_years']].sort_values('tenure_years', ascending=False))
    
    # Export
    output = io.StringIO()
    df.to_csv(output, index=False, encoding='utf-8')
    print("\n=== CSV Output ===")
    print(output.getvalue()[:200])
    
    # Export to JSON
    json_output = df.to_json(orient='records', date_format='iso', force_ascii=False, indent=2)
    print("\n=== JSON Output ===")
    print(json_output[:200])
```

---

## 4. Practical: Data Processing Pipeline

```python
import json
import csv
import io
from typing import List, Dict, Any, Optional
from datetime import datetime

class DataProcessor:
    """ระบบประมวลผลข้อมูลแบบ pipeline"""
    
    @staticmethod
    def from_json_string(json_str: str) -> List[Dict]:
        """อ่านจาก JSON string"""
        return json.loads(json_str)
    
    @staticmethod
    def from_json_file(filepath: str) -> List[Dict]:
        """อ่านจาก JSON file"""
        with open(filepath, 'r', encoding='utf-8') as f:
            return json.load(f)
    
    @staticmethod
    def from_csv_string(csv_str: str, delimiter: str = ',') -> List[Dict]:
        """อ่านจาก CSV string"""
        reader = csv.DictReader(io.StringIO(csv_str), delimiter=delimiter)
        return list(reader)
    
    @staticmethod
    def from_csv_file(filepath: str, delimiter: str = ',', 
                      encoding: str = 'utf-8-sig') -> List[Dict]:
        """อ่านจาก CSV file"""
        with open(filepath, 'r', encoding=encoding, newline='') as f:
            reader = csv.DictReader(f, delimiter=delimiter)
            return list(reader)
    
    @staticmethod
    def to_json_string(data: List[Dict], indent: int = 2) -> str:
        """แปลงเป็น JSON string"""
        return json.dumps(data, ensure_ascii=False, indent=indent, 
                         default=str)
    
    @staticmethod
    def to_csv_string(data: List[Dict], 
                      fieldnames: List[str] = None) -> str:
        """แปลงเป็น CSV string"""
        if not data:
            return ""
        
        if fieldnames is None:
            fieldnames = list(data[0].keys())
        
        output = io.StringIO()
        writer = csv.DictWriter(output, fieldnames=fieldnames)
        writer.writeheader()
        writer.writerows(data)
        return output.getvalue()
    
    @staticmethod
    def validate(data: List[Dict], 
                 required_fields: List[str],
                 validators: Dict[str, callable] = None) -> tuple:
        """
        ตรวจสอบความถูกต้องของข้อมูล
        
        Returns:
            (valid_records, invalid_records)
        """
        valid = []
        invalid = []
        validators = validators or {}
        
        for i, record in enumerate(data):
            errors = []
            
            # ตรวจสอบ required fields
            for field in required_fields:
                if field not in record or not record[field]:
                    errors.append(f"Missing required field: {field}")
            
            # ตรวจสอบ validators
            for field, validator in validators.items():
                if field in record:
                    try:
                        if not validator(record[field]):
                            errors.append(f"Invalid value for {field}: {record[field]}")
                    except Exception as e:
                        errors.append(f"Error validating {field}: {e}")
            
            if errors:
                invalid.append({
                    "record_index": i,
                    "record": record,
                    "errors": errors
                })
            else:
                valid.append(record)
        
        return valid, invalid
    
    @staticmethod
    def transform(data: List[Dict], 
                  transformations: Dict[str, callable]) -> List[Dict]:
        """แปลงข้อมูลตาม transformations"""
        result = []
        for record in data:
            new_record = record.copy()
            for field, func in transformations.items():
                if field in new_record:
                    try:
                        new_record[field] = func(new_record[field])
                    except Exception:
                        pass  # ข้าม error
            result.append(new_record)
        return result
    
    @staticmethod
    def merge(primary: List[Dict], secondary: List[Dict],
              on: str, how: str = 'inner') -> List[Dict]:
        """Merge สอง dataset"""
        secondary_index = {row[on]: row for row in secondary if on in row}
        
        result = []
        primary_keys = set()
        
        for primary_row in primary:
            if on not in primary_row:
                continue
                
            key = primary_row[on]
            primary_keys.add(key)
            
            if key in secondary_index:
                merged = {**primary_row, **secondary_index[key]}
                result.append(merged)
            elif how in ('left', 'outer'):
                result.append(primary_row.copy())
        
        if how in ('right', 'outer'):
            for key, secondary_row in secondary_index.items():
                if key not in primary_keys:
                    result.append(secondary_row.copy())
        
        return result


# ทดสอบ DataProcessor
# ข้อมูลพนักงาน (JSON)
employees_json = """[
    {"id": "E001", "name": "Alice", "dept_id": "D01", "salary": "75000"},
    {"id": "E002", "name": "Bob", "dept_id": "D02", "salary": "55000"},
    {"id": "E003", "name": "Charlie", "dept_id": "D01", "salary": "85000"},
    {"id": "E004", "name": "Dave", "dept_id": "D03", "salary": "abc"},
    {"id": "E005", "name": "", "dept_id": "D01", "salary": "90000"}
]"""

# ข้อมูลแผนก (CSV)
departments_csv = """dept_id,dept_name,budget
D01,Engineering,5000000
D02,Marketing,2000000
D03,HR,1500000
"""

# อ่านข้อมูล
employees = DataProcessor.from_json_string(employees_json)
departments = DataProcessor.from_csv_string(departments_csv)

# Validate
import re
valid_employees, invalid = DataProcessor.validate(
    employees,
    required_fields=["id", "name", "dept_id", "salary"],
    validators={
        "salary": lambda s: s.replace(".", "").isdigit(),
        "id": lambda i: bool(re.match(r'^E\d{3}$', i))
    }
)

print(f"Valid: {len(valid_employees)}, Invalid: {len(invalid)}")
for inv in invalid:
    print(f"  Record {inv['record_index']}: {inv['errors']}")

# Transform
transformed = DataProcessor.transform(
    valid_employees,
    {
        "salary": float,
        "name": str.strip,
    }
)

# Merge
merged = DataProcessor.merge(transformed, departments, on='dept_id')

print("\n=== Merged Data ===")
for row in merged:
    print(f"  {row['name']:10} | {row['dept_name']:12} | ฿{row['salary']:,.0f}")

# Export
print("\n=== JSON Output ===")
print(DataProcessor.to_json_string(merged[:2]))

print("\n=== CSV Output ===")
print(DataProcessor.to_csv_string(
    merged,
    fieldnames=["id", "name", "dept_name", "salary"]
))
```

---

## 5. Advanced JSON Operations

```python
import json
from typing import Any, Union

def flatten_json(data: dict, separator: str = '.', prefix: str = '') -> dict:
    """Flatten nested JSON เป็น flat dict"""
    result = {}
    
    for key, value in data.items():
        full_key = f"{prefix}{separator}{key}" if prefix else key
        
        if isinstance(value, dict):
            nested = flatten_json(value, separator, full_key)
            result.update(nested)
        elif isinstance(value, list):
            for i, item in enumerate(value):
                list_key = f"{full_key}[{i}]"
                if isinstance(item, dict):
                    nested = flatten_json(item, separator, list_key)
                    result.update(nested)
                else:
                    result[list_key] = item
        else:
            result[full_key] = value
    
    return result


def json_path(data: Any, path: str) -> Any:
    """
    ดึงค่าจาก JSON ด้วย path เช่น "user.address.city"
    หรือ "users[0].name"
    """
    import re
    
    parts = re.split(r'\.|\[(\d+)\]', path)
    current = data
    
    for part in parts:
        if not part:
            continue
        
        if isinstance(current, dict):
            current = current.get(part)
        elif isinstance(current, list):
            try:
                idx = int(part)
                current = current[idx]
            except (ValueError, IndexError):
                return None
        
        if current is None:
            return None
    
    return current


def merge_json(base: dict, override: dict) -> dict:
    """Merge สอง dict แบบ deep merge"""
    result = base.copy()
    
    for key, value in override.items():
        if key in result and isinstance(result[key], dict) and isinstance(value, dict):
            result[key] = merge_json(result[key], value)
        else:
            result[key] = value
    
    return result


# ทดสอบ
nested = {
    "user": {
        "id": 1,
        "name": "Alice",
        "address": {
            "street": "123 Main St",
            "city": "Bangkok",
            "country": "Thailand"
        }
    },
    "orders": [
        {"id": "ORD001", "total": 500},
        {"id": "ORD002", "total": 300},
    ]
}

flat = flatten_json(nested)
print("Flattened:")
for key, value in flat.items():
    print(f"  {key}: {value}")

print(f"\nPath query: user.name = {json_path(nested, 'user.name')}")
print(f"Path query: user.address.city = {json_path(nested, 'user.address.city')}")
print(f"Path query: orders[0].total = {json_path(nested, 'orders[0].total')}")

# Deep merge
config_base = {"db": {"host": "localhost", "port": 5432}, "debug": False}
config_override = {"db": {"port": 5433, "name": "mydb"}, "debug": True}
merged_config = merge_json(config_base, config_override)
print(f"\nMerged config: {merged_config}")
```

---

## Exercises

### Exercise 1: Student Grade System
สร้างระบบจัดการเกรดที่:
- อ่านข้อมูลนักเรียนจาก CSV
- อ่านเกรดจาก JSON
- Merge ข้อมูล
- Export รายงานเป็น CSV และ JSON

### Exercise 2: Config Manager
สร้าง config manager ที่:
- อ่าน config จาก JSON file
- รองรับ environment-specific overrides
- Deep merge config
- Validate config schema

### Exercise 3: CSV to JSON API
สร้าง converter ที่:
- รับ CSV input
- ตรวจสอบและ clean data
- แปลงเป็น JSON
- รองรับ type inference (string, number, boolean, date)

---

## สรุป

| Operation | JSON | CSV |
|-----------|------|-----|
| Parse string | `json.loads()` | `csv.reader()` |
| Parse file | `json.load()` | `csv.DictReader()` |
| Write string | `json.dumps()` | `csv.writer()` |
| Write file | `json.dump()` | `csv.DictWriter()` |
| Format | `indent=`, `sort_keys=` | `delimiter=`, `quoting=` |
| Custom types | `cls=CustomEncoder` | N/A |

**ข้อแนะนำ:**
- ใช้ `ensure_ascii=False` สำหรับ Unicode/ภาษาไทย
- ใช้ `newline=''` เมื่อเปิดไฟล์ CSV
- ใช้ `encoding='utf-8-sig'` สำหรับ CSV ที่เปิดใน Excel
- ใช้ pandas เมื่อต้องการ data analysis ขั้นสูง

---

## ต่อไป

[Part 025 - Virtual Environments and Package Management](part-025.md) - venv, pip, poetry, conda
