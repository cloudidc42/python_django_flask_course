# Part 021 - Regular Expressions (นิพจน์ปกติ)

## เป้าหมาย
- เข้าใจ Regular Expression metacharacters
- ใช้ `re` module: `match`, `search`, `findall`, `sub`, `split`
- ใช้ Groups และ Named Groups
- ใช้ Flags
- สร้าง patterns จริง: email, phone, URL, date
- ใช้ Lookahead และ Lookbehind

---

## 1. พื้นฐาน Regular Expressions

```python
import re

# ฟังก์ชันหลักใน re module
text = "Hello, my name is Alice and I am 25 years old."

# re.search() - ค้นหาแรก
match = re.search(r'\d+', text)
if match:
    print(match.group())   # 25
    print(match.start())   # index เริ่มต้น
    print(match.end())     # index สิ้นสุด
    print(match.span())    # (start, end)

# re.match() - ค้นหาที่ตำแหน่งเริ่มต้นเท่านั้น
m1 = re.match(r'Hello', text)   # Match! (ขึ้นต้นด้วย Hello)
m2 = re.match(r'Alice', text)   # None (ไม่ได้ขึ้นต้น)
print(m1)  # <re.Match object>
print(m2)  # None

# re.fullmatch() - ต้อง match ทั้งหมด
print(re.fullmatch(r'\d+', "12345"))    # Match
print(re.fullmatch(r'\d+', "123abc"))  # None

# re.findall() - หาทุกอัน คืน list
numbers = re.findall(r'\d+', "I have 3 cats and 2 dogs and 10 fish")
print(numbers)  # ['3', '2', '10']

# re.finditer() - หาทุกอัน คืน iterator ของ match objects
for m in re.finditer(r'\d+', "Items: 5 apples, 3 oranges, 12 bananas"):
    print(f"Found {m.group()} at position {m.start()}")

# re.sub() - แทนที่
result = re.sub(r'\d+', 'NUM', "I have 3 cats and 2 dogs")
print(result)  # I have NUM cats and NUM dogs

# re.sub กับ count
result = re.sub(r'\d+', 'NUM', "1 cat, 2 dogs, 3 fish", count=2)
print(result)  # NUM cat, NUM dogs, 3 fish

# re.split() - แยก string
parts = re.split(r'\s+', "Hello   World   Python")
print(parts)  # ['Hello', 'World', 'Python']

# re.split กับ delimiters หลายชนิด
parts = re.split(r'[,;:\s]+', "a, b; c:d  e")
print(parts)  # ['a', 'b', 'c', 'd', 'e']
```

---

## 2. Metacharacters และ Patterns

```python
import re

# === Character Classes ===
text = "Hello World 123 !@#"

# . - ทุก character ยกเว้น newline
print(re.findall(r'H.', text))  # ['He']

# \d - digit [0-9]
# \D - non-digit
print(re.findall(r'\d', text))   # ['1', '2', '3']
print(re.findall(r'\D', text))   # ['H', 'e', 'l', 'l', 'o', ...]

# \w - word character [a-zA-Z0-9_]
# \W - non-word character
print(re.findall(r'\w+', text))  # ['Hello', 'World', '123']

# \s - whitespace
# \S - non-whitespace
print(re.findall(r'\S+', text))  # ['Hello', 'World', '123', '!@#']

# \b - word boundary
print(re.findall(r'\bWorld\b', text))  # ['World']
print(re.findall(r'\b\w{5}\b', text)) # ['Hello', 'World']

# [] - character set
print(re.findall(r'[aeiou]', "Hello World"))  # ['e', 'o', 'o']
print(re.findall(r'[^aeiou\s]', "Hello World"))  # พยัญชนะ

# === Quantifiers ===
test = "aabbbcccc"

# * - 0 or more
print(re.findall(r'b*', "abc"))   # ['', '', 'b', '']

# + - 1 or more
print(re.findall(r'b+', "abbc"))  # ['bb']

# ? - 0 or 1 (optional)
print(re.findall(r'colou?r', "color colour"))  # ['color', 'colour']

# {n} - exactly n
print(re.findall(r'\d{3}', "12 123 1234 12345"))  # ['123', '123', '234']

# {n,m} - between n and m
print(re.findall(r'\d{2,4}', "1 12 123 1234 12345"))  # ['12', '123', '1234', '1234']

# {n,} - at least n
print(re.findall(r'\d{3,}', "1 12 123 1234"))  # ['123', '1234']

# === Anchors ===
# ^ - start of string
# $ - end of string
lines = ["Python is great", "Java is verbose", "Python rocks"]
python_lines = [l for l in lines if re.match(r'^Python', l)]
print(python_lines)  # ['Python is great', 'Python rocks']

# === Alternation ===
print(re.findall(r'cat|dog|fish', "I have a cat and a dog"))
# ['cat', 'dog']

# ตัวอย่าง pattern จริง
thai_phone = r'0[689]\d{8}'
phones = ["0812345678", "0912345678", "0612345678", "1234567890"]
valid_phones = [p for p in phones if re.fullmatch(thai_phone, p)]
print("Valid Thai phones:", valid_phones)
```

---

## 3. Groups

```python
import re

# Capturing Groups ()
text = "2024-10-15"
match = re.search(r'(\d{4})-(\d{2})-(\d{2})', text)
if match:
    print(match.group(0))  # 2024-10-15 (ทั้งหมด)
    print(match.group(1))  # 2024 (group 1)
    print(match.group(2))  # 10   (group 2)
    print(match.group(3))  # 15   (group 3)
    print(match.groups())  # ('2024', '10', '15')

# Named Groups (?P<name>...)
date_pattern = r'(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})'
match = re.search(date_pattern, "Date: 2024-10-15 is today")
if match:
    print(match.group("year"))   # 2024
    print(match.group("month"))  # 10
    print(match.group("day"))    # 15
    print(match.groupdict())     # {'year': '2024', 'month': '10', 'day': '15'}

# Non-capturing Groups (?:...)
# ใช้ grouping แต่ไม่ capture
text = "2024-10-15 and 2024/10/15"
pattern = r'\d{4}(?:-|/)\d{2}(?:-|/)\d{2}'
dates = re.findall(pattern, text)
print(dates)  # ['2024-10-15', '2024/10/15']

# Back References \1, \2 (อ้างอิง group ก่อนหน้า)
# หา word ที่ซ้ำ
text = "the the quick brown fox fox jumps"
duplicates = re.findall(r'\b(\w+)\b\s+\1\b', text)
print(duplicates)  # ['the', 'fox']

# re.sub กับ groups
text = "2024-10-15"
# เปลี่ยนรูปแบบวันที่จาก YYYY-MM-DD เป็น DD/MM/YYYY
result = re.sub(
    r'(\d{4})-(\d{2})-(\d{2})',
    r'\3/\2/\1',  # อ้างอิง group ด้วย \number
    text
)
print(result)  # 15/10/2024

# Named groups กับ sub
result = re.sub(
    r'(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})',
    r'\g<day>/\g<month>/\g<year>',  # \g<name>
    text
)
print(result)  # 15/10/2024

# Groups กับ findall
emails_text = "Contact: alice@example.com and bob@test.org"
pattern = r'(\w+)@(\w+)\.(\w+)'
matches = re.findall(pattern, emails_text)
print(matches)  # [('alice', 'example', 'com'), ('bob', 'test', 'org')]
```

---

## 4. Flags

```python
import re

text = """
Hello World
hello python
HELLO REGEX
Python is great
"""

# re.IGNORECASE (re.I) - case-insensitive
matches = re.findall(r'hello', text, re.IGNORECASE)
print(matches)  # ['Hello', 'hello', 'HELLO']

# re.MULTILINE (re.M) - ^ และ $ match ทุก line
# ไม่มี MULTILINE: ^ match แค่ start of string
# มี MULTILINE: ^ match start ของทุก line

# หา lines ที่ขึ้นต้นด้วย 'P' (case-sensitive)
lines_p = re.findall(r'^P.*', text, re.MULTILINE)
print(lines_p)  # ['Python is great']

# re.DOTALL (re.S) - . match newline ด้วย
html = "<p>First paragraph</p>\n<p>Second paragraph</p>"
# ไม่มี DOTALL
print(re.findall(r'<p>.*</p>', html))  # []
# มี DOTALL
print(re.findall(r'<p>.*</p>', html, re.DOTALL))  # ['<p>First...</p>']

# re.VERBOSE (re.X) - อนุญาต whitespace และ comments ใน pattern
email_pattern = re.compile(r"""
    \b                  # word boundary
    [a-zA-Z0-9._%+-]+  # username
    @                   # @ symbol
    [a-zA-Z0-9.-]+      # domain name
    \.                  # dot
    [a-zA-Z]{2,}        # TLD
    \b                  # word boundary
""", re.VERBOSE)

test_emails = ["alice@example.com", "not-an-email", "bob.smith@test.co.th"]
for email in test_emails:
    if email_pattern.search(email):
        print(f"Valid: {email}")
    else:
        print(f"Invalid: {email}")

# รวม flags ด้วย |
pattern = re.compile(r'hello\s+world', re.IGNORECASE | re.DOTALL)

# re.compile() - pre-compile pattern สำหรับ performance
compiled = re.compile(r'\d{4}-\d{2}-\d{2}')
dates = ["2024-01-15", "not-a-date", "2023-12-31"]
valid_dates = list(filter(compiled.fullmatch, dates))
print(valid_dates)
```

---

## 5. Lookahead และ Lookbehind

```python
import re

# Lookahead: (?=...) - positive lookahead
# (?!...) - negative lookahead

text = "apple 100px banana 200em cherry 300rem"

# Positive lookahead: ตัวเลขที่ตามด้วย px
px_values = re.findall(r'\d+(?=px)', text)
print(px_values)  # ['100']

# ตัวเลขที่ตามด้วย unit (px, em, rem)
any_unit = re.findall(r'\d+(?=px|em|rem)', text)
print(any_unit)  # ['100', '200', '300']

# Negative lookahead: ตัวเลขที่ไม่ตามด้วย px
not_px = re.findall(r'\d+(?!px)', text)
print(not_px)    # สังเกต: 100 -> '10' match เพราะ '0' ไม่ตามด้วย px

# ถูกต้อง: ตัวเลขที่ตามด้วย space หรือ end
isolated = re.findall(r'\b\d+\b(?!\w)', text)
print(isolated)  # []

# Lookbehind: (?<=...) - positive lookbehind
# (?<!...) - negative lookbehind

prices = "Regular: $100, Sale: $80, Member: $70"

# Positive lookbehind: ตัวเลขที่นำหน้าด้วย $
dollar_prices = re.findall(r'(?<=\$)\d+', prices)
print(dollar_prices)  # ['100', '80', '70']

# Negative lookbehind: ตัวเลขที่ไม่ได้นำหน้าด้วย $
non_dollar = re.findall(r'(?<!\$)\b\d+\b', prices)
print(non_dollar)  # []

# Lookahead ในการ validate password
def validate_password(password: str) -> dict:
    """
    Validate password:
    - ยาวอย่างน้อย 8 ตัว
    - มีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว
    - มีตัวพิมพ์เล็กอย่างน้อย 1 ตัว
    - มีตัวเลขอย่างน้อย 1 ตัว
    - มี special character อย่างน้อย 1 ตัว
    """
    checks = {
        "length": len(password) >= 8,
        "uppercase": bool(re.search(r'(?=.*[A-Z])', password)),
        "lowercase": bool(re.search(r'(?=.*[a-z])', password)),
        "digit": bool(re.search(r'(?=.*\d)', password)),
        "special": bool(re.search(r'(?=.*[!@#$%^&*(),.?":{}|<>])', password)),
    }
    checks["valid"] = all(checks.values())
    return checks

passwords = ["weak", "StrongPass1!", "NoDigitNoSpecial"]
for pwd in passwords:
    result = validate_password(pwd)
    print(f"'{pwd}': {'Valid' if result['valid'] else 'Invalid'}")
    if not result["valid"]:
        failed = [k for k, v in result.items() if not v and k != "valid"]
        print(f"  Missing: {', '.join(failed)}")
```

---

## 6. Practical Patterns

### Email Validation

```python
import re

def validate_email(email: str) -> bool:
    """Validate email address"""
    pattern = re.compile(
        r'^'
        r'[a-zA-Z0-9]'           # ตัวแรกต้องเป็น alphanumeric
        r'[a-zA-Z0-9._%+\-]*'    # ส่วนต่อไปของ username
        r'@'                      # @
        r'[a-zA-Z0-9\-]+'        # domain name
        r'(?:\.[a-zA-Z0-9\-]+)*' # sub-domains (optional)
        r'\.[a-zA-Z]{2,63}'      # TLD
        r'$',
        re.IGNORECASE
    )
    return bool(pattern.fullmatch(email))


def extract_emails(text: str) -> list:
    """ดึง email ออกจากข้อความ"""
    pattern = r'\b[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}\b'
    return re.findall(pattern, text)


# ทดสอบ
test_emails = [
    "alice@example.com",        # valid
    "bob.smith@test.co.uk",     # valid
    "user+tag@domain.org",      # valid
    "@invalid.com",             # invalid
    "no-at-sign",               # invalid
    "spaces @domain.com",       # invalid
    "user@.com",                # invalid
]

for email in test_emails:
    status = "✓" if validate_email(email) else "✗"
    print(f"{status} {email}")

# Extract from text
text = """
Contact us at info@company.com or support@help.org.
For sales: sales@company.com, john.doe@example.co.th
"""
print("\nExtracted emails:", extract_emails(text))
```

### Phone Number Patterns

```python
import re
from typing import Optional, Dict

def parse_thai_phone(phone: str) -> Optional[Dict[str, str]]:
    """Parse เบอร์โทรไทย"""
    # ล้าง whitespace และ dash
    cleaned = re.sub(r'[\s\-()]', '', phone)
    
    # Mobile: 08x, 09x, 06x (10 digits)
    mobile = re.fullmatch(r'(0[689]\d)(\d{3})(\d{4})', cleaned)
    if mobile:
        return {
            "type": "mobile",
            "formatted": f"{mobile.group(1)}-{mobile.group(2)}-{mobile.group(3)}",
            "original": phone
        }
    
    # Landline Bangkok: 02x (9 digits)
    landline_bkk = re.fullmatch(r'(02)(\d{3})(\d{4})', cleaned)
    if landline_bkk:
        return {
            "type": "landline_bangkok",
            "formatted": f"{landline_bkk.group(1)}-{landline_bkk.group(2)}-{landline_bkk.group(3)}",
            "original": phone
        }
    
    # Landline other: 03x-07x (9 digits)
    landline = re.fullmatch(r'(0[3-7]\d)(\d{3})(\d{3})', cleaned)
    if landline:
        return {
            "type": "landline",
            "formatted": f"{landline.group(1)}-{landline.group(2)}-{landline.group(3)}",
            "original": phone
        }
    
    return None


phones = [
    "0812345678",
    "081-234-5678",
    "081 234 5678",
    "(081) 234-5678",
    "025551234",
    "0455551234",
    "12345",  # invalid
]

for phone in phones:
    result = parse_thai_phone(phone)
    if result:
        print(f"Valid {result['type']}: {result['formatted']}")
    else:
        print(f"Invalid: {phone}")
```

### URL Pattern

```python
import re
from urllib.parse import urlparse
from typing import Dict, Optional

def parse_url(url: str) -> Optional[Dict[str, str]]:
    """Parse URL ด้วย regex"""
    pattern = re.compile(
        r'^'
        r'(?P<scheme>https?|ftp)'  # protocol
        r'://'
        r'(?P<host>'
            r'(?:(?:[a-zA-Z0-9](?:[a-zA-Z0-9\-]*[a-zA-Z0-9])?\.)+[a-zA-Z]{2,})'  # domain
            r'|localhost'            # or localhost
            r'|\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}'  # or IP
        r')'
        r'(?::(?P<port>\d+))?'     # optional port
        r'(?P<path>/[^?#]*)?'      # optional path
        r'(?:\?(?P<query>[^#]*))?'  # optional query
        r'(?:#(?P<fragment>.*))?'   # optional fragment
        r'$',
        re.IGNORECASE
    )
    
    match = pattern.fullmatch(url)
    if match:
        return match.groupdict()
    return None


def extract_urls(text: str) -> list:
    """ดึง URL ออกจากข้อความ"""
    pattern = r'https?://[^\s<>"]+[^\s<>",.]'
    return re.findall(pattern, text)


urls = [
    "https://www.example.com",
    "http://api.domain.com/users?page=1&limit=10#section",
    "https://localhost:8080/api/v1",
    "ftp://files.example.com/data.csv",
    "not-a-url",
    "http://192.168.1.1:3000/path",
]

for url in urls:
    result = parse_url(url)
    if result:
        print(f"Valid URL: {url}")
        for key, value in result.items():
            if value:
                print(f"  {key}: {value}")
    else:
        print(f"Invalid: {url}")
```

### Date Pattern

```python
import re
from datetime import datetime
from typing import Optional

def parse_date(date_str: str) -> Optional[datetime]:
    """Parse วันที่หลายรูปแบบ"""
    
    patterns = [
        # YYYY-MM-DD
        (r'(?P<year>\d{4})-(?P<month>\d{1,2})-(?P<day>\d{1,2})', 
         '%Y-%m-%d'),
        # DD/MM/YYYY
        (r'(?P<day>\d{1,2})/(?P<month>\d{1,2})/(?P<year>\d{4})',
         '%d/%m/%Y'),
        # DD-MM-YYYY
        (r'(?P<day>\d{1,2})-(?P<month>\d{1,2})-(?P<year>\d{4})',
         '%d-%m-%Y'),
        # Month name formats
        (r'(?P<day>\d{1,2})\s+(?P<month>Jan|Feb|Mar|Apr|May|Jun|Jul|Aug|Sep|Oct|Nov|Dec)\w*\s+(?P<year>\d{4})',
         None),
    ]
    
    for pattern, fmt in patterns:
        match = re.fullmatch(pattern, date_str.strip(), re.IGNORECASE)
        if match:
            d = match.groupdict()
            try:
                if fmt:
                    return datetime.strptime(date_str.strip(), fmt)
                else:
                    # Handle month names
                    month_map = {
                        'jan': 1, 'feb': 2, 'mar': 3, 'apr': 4,
                        'may': 5, 'jun': 6, 'jul': 7, 'aug': 8,
                        'sep': 9, 'oct': 10, 'nov': 11, 'dec': 12
                    }
                    month_str = d['month'][:3].lower()
                    return datetime(
                        int(d['year']),
                        month_map[month_str],
                        int(d['day'])
                    )
            except ValueError:
                pass
    
    return None


def extract_dates(text: str) -> list:
    """ดึงวันที่ออกจากข้อความ"""
    patterns = [
        r'\d{4}-\d{2}-\d{2}',
        r'\d{2}/\d{2}/\d{4}',
        r'\d{1,2}\s+(?:Jan|Feb|Mar|Apr|May|Jun|Jul|Aug|Sep|Oct|Nov|Dec)\w*\s+\d{4}',
    ]
    
    combined = '|'.join(f'({p})' for p in patterns)
    matches = re.findall(combined, text, re.IGNORECASE)
    
    # flatten results
    return [m for group in matches for m in group if m]


# ทดสอบ
dates_to_test = [
    "2024-10-15",
    "15/10/2024",
    "15-10-2024",
    "15 October 2024",
    "15 Oct 2024",
    "invalid-date",
    "2024-13-01",  # invalid month
]

for date_str in dates_to_test:
    result = parse_date(date_str)
    if result:
        print(f"'{date_str}' -> {result.strftime('%Y-%m-%d')}")
    else:
        print(f"'{date_str}' -> Invalid")
```

---

## 7. Advanced Patterns

### Text Cleaning

```python
import re
import unicodedata

def clean_text(text: str) -> str:
    """ทำความสะอาดข้อความ"""
    # ลบ HTML tags
    text = re.sub(r'<[^>]+>', '', text)
    
    # ลบ URL
    text = re.sub(r'https?://\S+', '', text)
    
    # แปลง multiple spaces เป็น single space
    text = re.sub(r'\s+', ' ', text)
    
    # ลบ special characters ที่ไม่ต้องการ
    text = re.sub(r'[^\w\s฀-๿.,!?;:()\-\'"]+', '', text)
    
    return text.strip()


def extract_structured_data(text: str) -> dict:
    """ดึงข้อมูล structured จาก text"""
    result = {}
    
    # Email
    email_match = re.search(r'[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}', text)
    if email_match:
        result["email"] = email_match.group()
    
    # Phone
    phone_match = re.search(r'0[689]\d{8}|0[2-7]\d{7}', re.sub(r'[\s\-]', '', text))
    if phone_match:
        result["phone"] = phone_match.group()
    
    # Price (บาท)
    price_match = re.search(r'(?:฿|บาท|THB)\s*(\d+(?:,\d{3})*(?:\.\d{2})?)', text)
    if price_match:
        result["price"] = float(price_match.group(1).replace(',', ''))
    
    # Date
    date_match = re.search(r'\d{4}-\d{2}-\d{2}', text)
    if date_match:
        result["date"] = date_match.group()
    
    return result


# ทดสอบ
sample = """
<html><body>
Contact John Doe at john.doe@example.com
Phone: 081-234-5678
Price: ฿1,299.99
Date: 2024-10-15
Visit our website: https://www.example.com
</body></html>
"""

cleaned = clean_text(sample)
print("Cleaned:", cleaned)
print()

data = extract_structured_data(sample)
print("Extracted data:", data)
```

### Log Parsing

```python
import re
from datetime import datetime
from typing import List, Dict, Optional

class LogParser:
    """Parser สำหรับ log files"""
    
    # Apache Combined Log Format
    APACHE_PATTERN = re.compile(
        r'(?P<ip>\d{1,3}(?:\.\d{1,3}){3})'  # IP address
        r'\s+\S+\s+\S+\s+'                   # ident, authuser
        r'\[(?P<datetime>[^\]]+)\]'           # datetime
        r'\s+"(?P<method>\w+)\s+'            # HTTP method
        r'(?P<path>[^\s"]+)'                  # path
        r'\s+HTTP/[\d.]+"\s+'                # protocol
        r'(?P<status>\d{3})\s+'              # status code
        r'(?P<size>\d+|-)'                   # response size
        r'(?:\s+"(?P<referer>[^"]*)")?'      # referer (optional)
        r'(?:\s+"(?P<user_agent>[^"]*)")?',  # user agent (optional)
        re.VERBOSE
    )
    
    @classmethod
    def parse_line(cls, line: str) -> Optional[Dict]:
        """Parse บรรทัดเดียว"""
        match = cls.APACHE_PATTERN.search(line)
        if not match:
            return None
        
        data = match.groupdict()
        
        # แปลงชนิดข้อมูล
        data["status"] = int(data["status"])
        data["size"] = int(data["size"]) if data["size"] != "-" else 0
        
        # Parse datetime
        try:
            data["datetime"] = datetime.strptime(
                data["datetime"], 
                "%d/%b/%Y:%H:%M:%S %z"
            )
        except ValueError:
            pass
        
        return data
    
    @classmethod
    def parse_file(cls, lines: List[str]) -> List[Dict]:
        """Parse หลายบรรทัด"""
        results = []
        for line in lines:
            parsed = cls.parse_line(line)
            if parsed:
                results.append(parsed)
        return results
    
    @staticmethod
    def summarize(logs: List[Dict]) -> Dict:
        """สรุปข้อมูล logs"""
        if not logs:
            return {}
        
        status_counts = {}
        for log in logs:
            status = log["status"]
            status_counts[status] = status_counts.get(status, 0) + 1
        
        return {
            "total_requests": len(logs),
            "status_counts": status_counts,
            "total_bytes": sum(log["size"] for log in logs),
            "unique_ips": len(set(log["ip"] for log in logs)),
            "error_rate": sum(1 for log in logs if log["status"] >= 400) / len(logs)
        }


# ทดสอบ
sample_logs = [
    '192.168.1.1 - - [15/Oct/2024:10:00:00 +0700] "GET /index.html HTTP/1.1" 200 1234 "-" "Mozilla/5.0"',
    '192.168.1.2 - - [15/Oct/2024:10:01:00 +0700] "POST /api/users HTTP/1.1" 201 256 "-" "curl/7.68.0"',
    '10.0.0.1 - - [15/Oct/2024:10:02:00 +0700] "GET /missing.html HTTP/1.1" 404 0 "-" "Chrome/90"',
    '192.168.1.1 - - [15/Oct/2024:10:03:00 +0700] "GET /error HTTP/1.1" 500 0 "-" "Firefox/80"',
]

parsed = LogParser.parse_file(sample_logs)
for log in parsed:
    print(f"{log['ip']} - {log['method']} {log['path']} - {log['status']}")

summary = LogParser.summarize(parsed)
print(f"\nSummary: {summary}")
```

---

## Exercises

### Exercise 1: HTML Parser
สร้าง function ที่ดึงข้อมูลจาก HTML:
- ดึง links ทั้งหมด (href)
- ดึง images (src, alt)
- ดึง text ออกจาก tags

### Exercise 2: Template Engine
สร้าง mini template engine:
```python
template = "Hello {{name}}, you have {{count}} messages"
render(template, {"name": "Alice", "count": 5})
# -> "Hello Alice, you have 5 messages"
```

### Exercise 3: Config File Parser
Parse config file format:
```ini
[database]
host = localhost
port = 5432
# ข้อความ comment

[app]
debug = true
```

---

## สรุป

| Pattern | ความหมาย | ตัวอย่าง |
|---------|---------|---------|
| `.` | ทุก char ยกเว้น `\n` | `a.c` matches "abc" |
| `\d` | digit | `\d+` matches "123" |
| `\w` | word char | `\w+` matches "hello_1" |
| `\s` | whitespace | `\s+` matches spaces |
| `*` | 0 or more | `a*` matches "", "a", "aaa" |
| `+` | 1 or more | `a+` matches "a", "aaa" |
| `?` | 0 or 1 | `colou?r` |
| `{n,m}` | n to m times | `\d{3,5}` |
| `()` | capture group | `(\d+)` |
| `(?P<n>)` | named group | `(?P<year>\d{4})` |
| `(?=...)` | lookahead | `\d+(?=px)` |
| `(?<=...)` | lookbehind | `(?<=\$)\d+` |
| `[abc]` | character set | `[aeiou]` |
| `[^abc]` | negated set | `[^\d]` |

---

## ต่อไป

[Part 022 - DateTime](part-022.md) - datetime module, timezone, formatting
