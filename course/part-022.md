# Part 022 - DateTime (วันและเวลา)

## เป้าหมาย
- เข้าใจ `datetime` module: date, time, datetime, timedelta
- Formatting ด้วย `strftime` และ `strptime`
- จัดการ Timezone ด้วย `pytz` และ `zoneinfo`
- แนะนำ `arrow` library
- ตัวอย่างจริง: อายุ, ตารางเวลา, deadline

---

## 1. datetime Module พื้นฐาน

```python
from datetime import date, time, datetime, timedelta

# === date ===
today = date.today()
print(today)           # 2024-10-15
print(today.year)      # 2024
print(today.month)     # 10
print(today.day)       # 15
print(today.weekday()) # 1 (0=Monday, 6=Sunday)
print(today.isoformat())  # '2024-10-15'

# สร้าง date เฉพาะ
birthday = date(1990, 5, 15)
print(birthday)  # 1990-05-15

# date arithmetic
tomorrow = today + timedelta(days=1)
yesterday = today - timedelta(days=1)
next_week = today + timedelta(weeks=1)

diff = today - birthday
print(f"Days since birthday: {diff.days:,}")

# === time ===
current_time = time(14, 30, 0)  # 14:30:00
print(current_time)           # 14:30:00
print(current_time.hour)      # 14
print(current_time.minute)    # 30
print(current_time.second)    # 0
print(current_time.isoformat()) # '14:30:00'

# time with microseconds
precise_time = time(14, 30, 45, 123456)  # 14:30:45.123456
print(precise_time)  # 14:30:45.123456

# === datetime ===
now = datetime.now()           # วันเวลาปัจจุบัน (local)
utc_now = datetime.utcnow()    # UTC (deprecated ใน Python 3.12)

print(now)           # 2024-10-15 14:30:45.123456
print(now.date())    # 2024-10-15
print(now.time())    # 14:30:45.123456

# สร้าง datetime เฉพาะ
event = datetime(2024, 12, 31, 23, 59, 59)
print(event)  # 2024-12-31 23:59:59

# Combine date และ time
d = date(2024, 10, 15)
t = time(14, 30, 0)
dt = datetime.combine(d, t)
print(dt)  # 2024-10-15 14:30:00

# datetime min, max
print(datetime.min)  # 0001-01-01 00:00:00
print(datetime.max)  # 9999-12-31 23:59:59.999999

# === timedelta ===
delta = timedelta(
    days=7,
    hours=2,
    minutes=30,
    seconds=15,
    microseconds=500
)
print(delta)
print(delta.days)         # 7
print(delta.seconds)      # 9015 (2*3600 + 30*60 + 15)
print(delta.total_seconds())  # 616815.0005

# timedelta arithmetic
start = datetime(2024, 1, 1)
end = datetime(2024, 12, 31)
duration = end - start
print(f"Days in 2024: {duration.days}")  # 365
```

---

## 2. Formatting: strftime และ strptime

```python
from datetime import datetime

now = datetime(2024, 10, 15, 14, 30, 45)

# strftime - datetime -> string
# Format codes:
print(now.strftime("%Y-%m-%d"))           # 2024-10-15
print(now.strftime("%d/%m/%Y"))           # 15/10/2024
print(now.strftime("%d %B %Y"))           # 15 October 2024
print(now.strftime("%d %b %Y"))           # 15 Oct 2024
print(now.strftime("%A, %d %B %Y"))       # Tuesday, 15 October 2024
print(now.strftime("%I:%M %p"))           # 02:30 PM
print(now.strftime("%H:%M:%S"))           # 14:30:45
print(now.strftime("%Y-%m-%dT%H:%M:%S"))  # 2024-10-15T14:30:45

# Format codes ที่ใช้บ่อย
format_codes = {
    "%Y": "4-digit year (2024)",
    "%m": "2-digit month (01-12)",
    "%d": "2-digit day (01-31)",
    "%H": "24-hour hour (00-23)",
    "%I": "12-hour hour (01-12)",
    "%M": "minute (00-59)",
    "%S": "second (00-59)",
    "%f": "microsecond (000000-999999)",
    "%p": "AM/PM",
    "%A": "Full weekday name",
    "%a": "Abbreviated weekday",
    "%B": "Full month name",
    "%b": "Abbreviated month name",
    "%j": "Day of year (001-366)",
    "%W": "Week number (00-53)",
    "%Z": "Timezone name",
    "%z": "UTC offset (+0700)",
    "%%": "Literal %",
}

# strptime - string -> datetime
date_strings = [
    ("2024-10-15", "%Y-%m-%d"),
    ("15/10/2024", "%d/%m/%Y"),
    ("15 October 2024", "%d %B %Y"),
    ("Oct 15, 2024 02:30 PM", "%b %d, %Y %I:%M %p"),
]

for date_str, fmt in date_strings:
    dt = datetime.strptime(date_str, fmt)
    print(f"'{date_str}' -> {dt}")

# ISO 8601 format
iso_str = "2024-10-15T14:30:45"
dt = datetime.fromisoformat(iso_str)
print(dt)  # 2024-10-15 14:30:45

# Python 3.11+ รองรับ Z suffix
# dt = datetime.fromisoformat("2024-10-15T14:30:45Z")


# Custom Thai date format
def format_thai_date(dt: datetime) -> str:
    """แสดงวันที่ภาษาไทย"""
    thai_months = [
        "มกราคม", "กุมภาพันธ์", "มีนาคม", "เมษายน",
        "พฤษภาคม", "มิถุนายน", "กรกฎาคม", "สิงหาคม",
        "กันยายน", "ตุลาคม", "พฤศจิกายน", "ธันวาคม"
    ]
    thai_days = [
        "จันทร์", "อังคาร", "พุธ", "พฤหัสบดี",
        "ศุกร์", "เสาร์", "อาทิตย์"
    ]
    
    day_name = thai_days[dt.weekday()]
    month_name = thai_months[dt.month - 1]
    thai_year = dt.year + 543  # ปี พ.ศ.
    
    return f"วัน{day_name}ที่ {dt.day} {month_name} พ.ศ. {thai_year}"


print(format_thai_date(datetime(2024, 10, 15)))
# วันอังคารที่ 15 ตุลาคม พ.ศ. 2567
```

---

## 3. Timezone

### ใช้ zoneinfo (Python 3.9+)

```python
from datetime import datetime
from zoneinfo import ZoneInfo, available_timezones

# Timezone-aware datetime
bangkok_tz = ZoneInfo("Asia/Bangkok")
utc_tz = ZoneInfo("UTC")
tokyo_tz = ZoneInfo("Asia/Tokyo")
london_tz = ZoneInfo("Europe/London")
la_tz = ZoneInfo("America/Los_Angeles")

# สร้าง timezone-aware datetime
now_bangkok = datetime.now(bangkok_tz)
now_utc = datetime.now(utc_tz)
print(f"Bangkok: {now_bangkok.strftime('%Y-%m-%d %H:%M:%S %Z')}")
print(f"UTC:     {now_utc.strftime('%Y-%m-%d %H:%M:%S %Z')}")

# แปลง timezone (convert)
now_tokyo = now_bangkok.astimezone(tokyo_tz)
now_london = now_bangkok.astimezone(london_tz)
now_la = now_bangkok.astimezone(la_tz)

print(f"Tokyo:   {now_tokyo.strftime('%Y-%m-%d %H:%M:%S %Z')}")
print(f"London:  {now_london.strftime('%Y-%m-%d %H:%M:%S %Z')}")
print(f"LA:      {now_la.strftime('%Y-%m-%d %H:%M:%S %Z')}")

# Localize naive datetime (ไม่มี tz info)
naive_dt = datetime(2024, 10, 15, 14, 30, 0)  # naive
aware_dt = naive_dt.replace(tzinfo=bangkok_tz)  # ใส่ tz info
print(f"Naive: {naive_dt}")
print(f"Aware: {aware_dt}")

# ระวัง: replace != convert
# replace บอกว่า "นี่คือเวลา Bangkok"
# astimezone บอกว่า "แปลงจาก timezone นี้ไป timezone นั้น"

# ดู available timezones
asian_tz = [tz for tz in available_timezones() if "Asia" in tz]
print(f"Asian timezones: {sorted(asian_tz)[:5]}")
```

### ใช้ pytz (สำหรับ Python < 3.9)

```python
try:
    import pytz
    HAS_PYTZ = True
except ImportError:
    HAS_PYTZ = False
    print("pytz not installed, using zoneinfo instead")

if HAS_PYTZ:
    from datetime import datetime
    import pytz
    
    # สร้าง timezone objects
    bangkok = pytz.timezone("Asia/Bangkok")
    utc = pytz.UTC
    tokyo = pytz.timezone("Asia/Tokyo")
    
    # สร้าง timezone-aware datetime
    now_utc = datetime.now(utc)
    
    # แปลง timezone
    now_bangkok = now_utc.astimezone(bangkok)
    now_tokyo = now_utc.astimezone(tokyo)
    
    print(f"UTC:     {now_utc.strftime('%Y-%m-%d %H:%M:%S %Z')}")
    print(f"Bangkok: {now_bangkok.strftime('%Y-%m-%d %H:%M:%S %Z')}")
    print(f"Tokyo:   {now_tokyo.strftime('%Y-%m-%d %H:%M:%S %Z')}")
    
    # Localize naive datetime
    naive = datetime(2024, 10, 15, 14, 30, 0)
    localized = bangkok.localize(naive)  # ใช้ localize() ไม่ใช่ replace()
    print(f"Localized: {localized}")

# UTC offset
from datetime import timezone, timedelta

# สร้าง timezone ด้วย offset
bkk_tz = timezone(timedelta(hours=7), name="ICT")  # Bangkok UTC+7
jst_tz = timezone(timedelta(hours=9), name="JST")   # Japan UTC+9

dt = datetime(2024, 10, 15, 12, 0, 0, tzinfo=bkk_tz)
print(dt)
print(dt.utcoffset())  # 7:00:00
```

---

## 4. Practical Examples

### คำนวณอายุ

```python
from datetime import date, datetime
from typing import Tuple

def calculate_age(birth_date: date, reference_date: date = None) -> dict:
    """
    คำนวณอายุอย่างละเอียด
    
    Returns:
        dict ที่มี years, months, days
    """
    if reference_date is None:
        reference_date = date.today()
    
    if birth_date > reference_date:
        raise ValueError("Birth date cannot be in the future")
    
    years = reference_date.year - birth_date.year
    months = reference_date.month - birth_date.month
    days = reference_date.day - birth_date.day
    
    if days < 0:
        months -= 1
        # หาวันในเดือนก่อนหน้า
        import calendar
        prev_month = reference_date.month - 1 if reference_date.month > 1 else 12
        prev_year = reference_date.year if reference_date.month > 1 else reference_date.year - 1
        days_in_prev_month = calendar.monthrange(prev_year, prev_month)[1]
        days += days_in_prev_month
    
    if months < 0:
        years -= 1
        months += 12
    
    total_days = (reference_date - birth_date).days
    
    return {
        "years": years,
        "months": months,
        "days": days,
        "total_days": total_days,
        "is_birthday": (birth_date.month == reference_date.month and 
                       birth_date.day == reference_date.day)
    }


def next_birthday(birth_date: date) -> Tuple[date, int]:
    """คำนวณวันเกิดครั้งถัดไปและอีกกี่วัน"""
    today = date.today()
    next_bday = birth_date.replace(year=today.year)
    
    if next_bday < today:
        next_bday = birth_date.replace(year=today.year + 1)
    
    days_until = (next_bday - today).days
    return next_bday, days_until


# ทดสอบ
birth = date(1990, 5, 15)
age = calculate_age(birth)
print(f"Age: {age['years']} years, {age['months']} months, {age['days']} days")
print(f"Total days lived: {age['total_days']:,}")
print(f"Is birthday today: {age['is_birthday']}")

next_bday, days = next_birthday(birth)
print(f"Next birthday: {next_bday} (in {days} days)")
```

### Business Hours Calculator

```python
from datetime import datetime, time, timedelta, date
from typing import List, Tuple
import calendar

class BusinessHoursCalculator:
    """คำนวณเวลาทำการ"""
    
    def __init__(self, 
                 work_start: time = time(9, 0),
                 work_end: time = time(18, 0),
                 work_days: List[int] = None,
                 holidays: List[date] = None):
        """
        Args:
            work_start: เวลาเริ่มงาน
            work_end: เวลาเลิกงาน
            work_days: วันทำงาน (0=Mon, 6=Sun) default: Mon-Fri
            holidays: วันหยุดพิเศษ
        """
        self.work_start = work_start
        self.work_end = work_end
        self.work_days = work_days or [0, 1, 2, 3, 4]  # Mon-Fri
        self.holidays = set(holidays or [])
        
        # คำนวณชั่วโมงทำงานต่อวัน
        start_seconds = work_start.hour * 3600 + work_start.minute * 60
        end_seconds = work_end.hour * 3600 + work_end.minute * 60
        self.work_seconds_per_day = end_seconds - start_seconds
    
    def is_working_day(self, dt: date) -> bool:
        """ตรวจสอบว่าเป็นวันทำงานหรือไม่"""
        return (dt.weekday() in self.work_days and 
                dt not in self.holidays)
    
    def is_working_time(self, dt: datetime) -> bool:
        """ตรวจสอบว่าเป็นเวลาทำการหรือไม่"""
        if not self.is_working_day(dt.date()):
            return False
        return self.work_start <= dt.time() < self.work_end
    
    def next_working_datetime(self, dt: datetime) -> datetime:
        """หาเวลาทำการถัดไป"""
        if self.is_working_time(dt):
            return dt
        
        # ถ้าอยู่ในวันทำงานแต่นอกเวลา
        if (self.is_working_day(dt.date()) and 
            dt.time() < self.work_start):
            return datetime.combine(dt.date(), self.work_start)
        
        # หาวันทำงานถัดไป
        next_day = dt.date() + timedelta(days=1)
        while not self.is_working_day(next_day):
            next_day += timedelta(days=1)
        
        return datetime.combine(next_day, self.work_start)
    
    def add_working_hours(self, start: datetime, hours: float) -> datetime:
        """เพิ่มชั่วโมงทำงาน"""
        if not self.is_working_time(start):
            start = self.next_working_datetime(start)
        
        remaining_seconds = hours * 3600
        current = start
        
        while remaining_seconds > 0:
            # เวลาที่เหลือในวันนี้
            end_today = datetime.combine(current.date(), self.work_end)
            available_seconds = (end_today - current).total_seconds()
            
            if remaining_seconds <= available_seconds:
                current += timedelta(seconds=remaining_seconds)
                break
            else:
                remaining_seconds -= available_seconds
                # ไปวันทำงานถัดไป
                next_day = current.date() + timedelta(days=1)
                while not self.is_working_day(next_day):
                    next_day += timedelta(days=1)
                current = datetime.combine(next_day, self.work_start)
        
        return current
    
    def business_hours_between(self, start: datetime, end: datetime) -> float:
        """คำนวณชั่วโมงทำงานระหว่างสองเวลา"""
        if start >= end:
            return 0
        
        total_seconds = 0
        current = start
        
        while current < end:
            if self.is_working_day(current.date()):
                work_start_dt = datetime.combine(current.date(), self.work_start)
                work_end_dt = datetime.combine(current.date(), self.work_end)
                
                # คำนวณ overlap กับเวลาทำงาน
                effective_start = max(current, work_start_dt)
                effective_end = min(end, work_end_dt)
                
                if effective_start < effective_end:
                    total_seconds += (effective_end - effective_start).total_seconds()
            
            # ไปวันถัดไป
            next_date = current.date() + timedelta(days=1)
            current = datetime.combine(next_date, self.work_start)
        
        return total_seconds / 3600
    
    def working_days_between(self, start: date, end: date) -> int:
        """นับวันทำงานระหว่างสองวัน"""
        count = 0
        current = start
        while current < end:
            if self.is_working_day(current):
                count += 1
            current += timedelta(days=1)
        return count


# ทดสอบ
from datetime import date, datetime, time

# วันหยุดราชการ 2024
thai_holidays_2024 = [
    date(2024, 1, 1),   # วันขึ้นปีใหม่
    date(2024, 2, 26),  # วันมาฆบูชา
    date(2024, 4, 6),   # วันจักรี
    date(2024, 4, 13),  # วันสงกรานต์
    date(2024, 5, 1),   # วันแรงงาน
    date(2024, 5, 22),  # วันวิสาขบูชา
    date(2024, 6, 3),   # วันเฉลิมพระชนมพรรษา
    date(2024, 7, 22),  # วันอาสาฬหบูชา
    date(2024, 7, 23),  # วันเข้าพรรษา
    date(2024, 8, 12),  # วันแม่แห่งชาติ
    date(2024, 10, 13), # วันนวมินทรมหาราช
    date(2024, 10, 23), # วันปิยมหาราช
    date(2024, 12, 5),  # วันพ่อแห่งชาติ
    date(2024, 12, 10), # วันรัฐธรรมนูญ
    date(2024, 12, 31), # วันสิ้นปี
]

calc = BusinessHoursCalculator(
    work_start=time(9, 0),
    work_end=time(18, 0),
    holidays=thai_holidays_2024
)

# สั่งงานวันศุกร์บ่ายโมง จะเสร็จเมื่อไหร่ถ้าต้องใช้เวลา 10 ชั่วโมง
start = datetime(2024, 10, 11, 14, 0, 0)  # Friday 2 PM
deadline = calc.add_working_hours(start, 10)
print(f"Start: {start}")
print(f"10 business hours later: {deadline}")

# นับวันทำงาน
work_days = calc.working_days_between(
    date(2024, 10, 1),
    date(2024, 10, 31)
)
print(f"Working days in October 2024: {work_days}")
```

### Scheduling

```python
from datetime import datetime, timedelta, timezone
from zoneinfo import ZoneInfo
from typing import List, Dict, Optional
import calendar

class Scheduler:
    """ตัวจัดการ schedule"""
    
    def __init__(self, timezone_str: str = "Asia/Bangkok"):
        self.tz = ZoneInfo(timezone_str)
    
    def now(self) -> datetime:
        """เวลาปัจจุบันใน timezone"""
        return datetime.now(self.tz)
    
    def recurring_dates(self, 
                        start: datetime,
                        recurrence: str,
                        count: int = 10) -> List[datetime]:
        """
        สร้าง recurring dates
        
        Args:
            recurrence: 'daily', 'weekly', 'monthly', 'yearly'
        """
        dates = [start]
        current = start
        
        for _ in range(count - 1):
            if recurrence == 'daily':
                current += timedelta(days=1)
            elif recurrence == 'weekly':
                current += timedelta(weeks=1)
            elif recurrence == 'monthly':
                # เพิ่มหนึ่งเดือน (จัดการกรณี 30/31 วัน)
                if current.month == 12:
                    current = current.replace(
                        year=current.year + 1, month=1)
                else:
                    # ดูแลกรณี 31 Jan + 1 month = ?
                    next_month = current.month + 1
                    max_day = calendar.monthrange(current.year, next_month)[1]
                    day = min(current.day, max_day)
                    current = current.replace(month=next_month, day=day)
            elif recurrence == 'yearly':
                # เพิ่มหนึ่งปี
                try:
                    current = current.replace(year=current.year + 1)
                except ValueError:  # Feb 29 ในปีไม่ใช่ leap year
                    current = current.replace(
                        year=current.year + 1, day=28)
            
            dates.append(current)
        
        return dates
    
    def countdown(self, target: datetime) -> Dict:
        """นับถอยหลังไปยัง datetime"""
        now = self.now()
        
        if target <= now:
            return {"expired": True, "total_seconds": 0}
        
        diff = target - now
        total_seconds = int(diff.total_seconds())
        
        days = total_seconds // 86400
        remaining = total_seconds % 86400
        hours = remaining // 3600
        remaining = remaining % 3600
        minutes = remaining // 60
        seconds = remaining % 60
        
        return {
            "expired": False,
            "days": days,
            "hours": hours,
            "minutes": minutes,
            "seconds": seconds,
            "total_seconds": total_seconds,
            "formatted": f"{days}d {hours:02d}h {minutes:02d}m {seconds:02d}s"
        }
    
    def humanize_duration(self, seconds: int) -> str:
        """แปลง duration เป็นภาษาคน"""
        if seconds < 60:
            return f"{seconds} วินาที"
        elif seconds < 3600:
            minutes = seconds // 60
            secs = seconds % 60
            return f"{minutes} นาที {secs} วินาที" if secs else f"{minutes} นาที"
        elif seconds < 86400:
            hours = seconds // 3600
            mins = (seconds % 3600) // 60
            return f"{hours} ชั่วโมง {mins} นาที" if mins else f"{hours} ชั่วโมง"
        else:
            days = seconds // 86400
            hours = (seconds % 86400) // 3600
            return f"{days} วัน {hours} ชั่วโมง" if hours else f"{days} วัน"


# ทดสอบ
bkk_tz = ZoneInfo("Asia/Bangkok")
scheduler = Scheduler("Asia/Bangkok")

# Recurring events
start = datetime(2024, 10, 1, 9, 0, tzinfo=bkk_tz)
monthly = scheduler.recurring_dates(start, 'monthly', 3)
print("Monthly events:")
for event in monthly:
    print(f"  {event.strftime('%Y-%m-%d %H:%M %Z')}")

# Countdown
new_year = datetime(2025, 1, 1, 0, 0, 0, tzinfo=bkk_tz)
countdown = scheduler.countdown(new_year)
print(f"\nCountdown to New Year: {countdown['formatted']}")

# Humanize
for secs in [45, 125, 3700, 90000]:
    print(f"{secs}s = {scheduler.humanize_duration(secs)}")
```

---

## 5. arrow Library

```python
# pip install arrow
# arrow เป็น library ที่ใช้งานง่ายกว่า datetime

try:
    import arrow
    
    # สร้าง arrow objects
    now = arrow.now()                     # local time
    now_bkk = arrow.now("Asia/Bangkok")   # Bangkok time
    utc_now = arrow.utcnow()              # UTC
    
    print(now_bkk.format("YYYY-MM-DD HH:mm:ss ZZ"))
    
    # สร้างจาก string
    dt = arrow.get("2024-10-15", "YYYY-MM-DD")
    dt2 = arrow.get("2024-10-15T14:30:00")
    
    # แปลง timezone
    bkk = arrow.now("Asia/Bangkok")
    tokyo = bkk.to("Asia/Tokyo")
    london = bkk.to("Europe/London")
    
    print(f"Bangkok: {bkk.format('HH:mm ZZZ')}")
    print(f"Tokyo:   {tokyo.format('HH:mm ZZZ')}")
    print(f"London:  {london.format('HH:mm ZZZ')}")
    
    # Humanize (เวลาสัมพัทธ์)
    past = arrow.now().shift(hours=-2, minutes=-30)
    print(past.humanize())  # "2 hours ago"
    
    future = arrow.now().shift(days=3)
    print(future.humanize())  # "in 3 days"
    
    # Shift (เพิ่ม/ลด)
    dt = arrow.get("2024-10-15")
    print(dt.shift(days=7).format("YYYY-MM-DD"))   # 2024-10-22
    print(dt.shift(months=1).format("YYYY-MM-DD")) # 2024-11-15
    print(dt.shift(years=-1).format("YYYY-MM-DD")) # 2023-10-15
    
    # Ranges
    start = arrow.get("2024-10-01")
    end = arrow.get("2024-10-31")
    
    for dt in arrow.Arrow.range("week", start, end):
        print(dt.format("YYYY-MM-DD (dddd)"))
    
    # Floor and ceiling
    dt = arrow.get("2024-10-15 14:37:22")
    print(dt.floor("day"))   # 2024-10-15 00:00:00+00:00
    print(dt.ceil("hour"))   # 2024-10-15 15:00:00+00:00
    
    HAS_ARROW = True
    
except ImportError:
    print("arrow not installed. Install with: pip install arrow")
    HAS_ARROW = False
```

---

## 6. Parsing Dates จาก String

```python
from datetime import datetime
import re

def smart_date_parse(date_str: str) -> datetime:
    """
    Parse วันที่จาก string รูปแบบต่างๆ
    """
    # ลอง formats ทั่วไป
    formats = [
        "%Y-%m-%d",
        "%d/%m/%Y",
        "%m/%d/%Y",
        "%d-%m-%Y",
        "%Y/%m/%d",
        "%d %B %Y",
        "%d %b %Y",
        "%B %d, %Y",
        "%b %d, %Y",
        "%Y-%m-%dT%H:%M:%S",
        "%Y-%m-%d %H:%M:%S",
        "%d/%m/%Y %H:%M",
    ]
    
    cleaned = date_str.strip()
    
    for fmt in formats:
        try:
            return datetime.strptime(cleaned, fmt)
        except ValueError:
            continue
    
    # ลอง regex สำหรับ relative dates
    relative_patterns = {
        r"today": lambda: datetime.now(),
        r"yesterday": lambda: datetime.now().replace(hour=0, minute=0, second=0) - __import__('datetime').timedelta(days=1),
        r"tomorrow": lambda: datetime.now().replace(hour=0, minute=0, second=0) + __import__('datetime').timedelta(days=1),
    }
    
    for pattern, func in relative_patterns.items():
        if re.match(pattern, cleaned, re.IGNORECASE):
            return func()
    
    raise ValueError(f"Cannot parse date: {date_str!r}")


# ทดสอบ
date_strings = [
    "2024-10-15",
    "15/10/2024",
    "15 October 2024",
    "October 15, 2024",
    "2024-10-15T14:30:00",
    "today",
]

for ds in date_strings:
    try:
        result = smart_date_parse(ds)
        print(f"'{ds}' -> {result.strftime('%Y-%m-%d %H:%M')}")
    except ValueError as e:
        print(f"Error parsing '{ds}': {e}")
```

---

## Exercises

### Exercise 1: Age Calculator CLI
สร้าง program ที่รับ birthdate และแสดง:
- อายุ (ปี, เดือน, วัน)
- วันเกิดครั้งถัดไป
- วันในสัปดาห์ที่เกิด
- ราศี

### Exercise 2: Meeting Scheduler
สร้าง system ที่:
- รับเวลาว่างของแต่ละคน
- หา time slot ที่ทุกคนว่าง
- แสดงผลใน timezone ของแต่ละคน

### Exercise 3: Deadline Tracker
สร้าง tracker ที่:
- เก็บ tasks กับ deadline
- แสดงว่าเหลือเวลาอีกกี่ชั่วโมงทำงาน
- เตือนเมื่อ deadline ใกล้

---

## สรุป

| Class/Method | คำอธิบาย | ตัวอย่าง |
|-------------|---------|---------|
| `date` | วันที่ | `date(2024, 10, 15)` |
| `time` | เวลา | `time(14, 30, 0)` |
| `datetime` | วันและเวลา | `datetime.now()` |
| `timedelta` | ช่วงเวลา | `timedelta(days=7)` |
| `strftime` | datetime -> string | `dt.strftime("%Y-%m-%d")` |
| `strptime` | string -> datetime | `datetime.strptime(s, fmt)` |
| `ZoneInfo` | timezone | `ZoneInfo("Asia/Bangkok")` |
| `astimezone` | แปลง timezone | `dt.astimezone(tz)` |
| `arrow` | library สะดวก | `arrow.now().humanize()` |

---

## ต่อไป

[Part 023 - Math and Random](part-023.md) - math, statistics, random, numpy intro
