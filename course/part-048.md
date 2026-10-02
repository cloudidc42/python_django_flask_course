# Part 048: Performance Optimization
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- ใช้ `cProfile` และ `pstats` วิเคราะห์ประสิทธิภาพโปรแกรม
- ใช้ `line_profiler` วิเคราะห์แบบบรรทัดต่อบรรทัด
- ใช้ `memory_profiler` ตรวจสอบการใช้หน่วยความจำ
- เพิ่มประสิทธิภาพด้วย `functools.lru_cache` และ `cache`
- ใช้ Redis caching ด้วย `redis-py`
- เข้าใจ Lazy evaluation และ Generators
- หลีกเลี่ยงปัญหาที่พบบ่อยที่ทำให้โปรแกรมช้า
- วัดประสิทธิภาพด้วย `timeit`

---

## 1. Profiling ด้วย cProfile

`cProfile` เป็น built-in profiler ของ Python ที่วัดว่าแต่ละ function ใช้เวลาเท่าไหร่

### 1.1 การใช้งาน cProfile พื้นฐาน

```python
import cProfile
import pstats
from pstats import SortKey
import io

# === ตัวอย่างโปรแกรมที่ต้องการ profile ===
def slow_function():
    """ฟังก์ชันที่ช้าเพราะใช้ list comprehension ซ้ำๆ"""
    result = []
    for i in range(10000):
        # ปัญหา: สร้าง list ใหม่ทุกครั้ง
        result = [x * x for x in range(i)]
    return result[-1] if result else 0

def fast_function():
    """ฟังก์ชันที่เร็วขึ้น"""
    last = 0
    for i in range(10000):
        # แก้ไข: คำนวณเฉพาะค่าสุดท้าย
        if i > 0:
            last = (i - 1) * (i - 1)
    return last

def fibonacci_naive(n):
    """Fibonacci แบบ recursive ที่ไม่มี cache — ช้ามาก"""
    if n <= 1:
        return n
    return fibonacci_naive(n - 1) + fibonacci_naive(n - 2)

def fibonacci_dp(n):
    """Fibonacci ด้วย dynamic programming — เร็วกว่ามาก"""
    if n <= 1:
        return n
    a, b = 0, 1
    for _ in range(2, n + 1):
        a, b = b, a + b
    return b

# === วิธีที่ 1: Profile โดยตรงด้วย cProfile.run() ===
print("=== Profile slow_function ===")
cProfile.run("slow_function()")

# === วิธีที่ 2: Profile และเก็บผลลัพธ์ ===
print("\n=== Profile พร้อม pstats ===")
profiler = cProfile.Profile()
profiler.enable()           # เริ่ม profiling

# โค้ดที่ต้องการ profile
slow_function()
fibonacci_naive(30)

profiler.disable()          # หยุด profiling

# แสดงผลลัพธ์
stats = pstats.Stats(profiler)
stats.sort_stats(SortKey.CUMULATIVE)     # เรียงตามเวลาสะสม
stats.print_stats(20)                    # แสดง 20 อันดับแรก

# === วิธีที่ 3: บันทึกผลลัพธ์ลงไฟล์ ===
profiler.dump_stats("profile_output.prof")

# อ่านและแสดงผลจากไฟล์
stats = pstats.Stats("profile_output.prof")
stats.strip_dirs()                       # ลบ path ออกเพื่อความสะอาด
stats.sort_stats(SortKey.TIME)           # เรียงตามเวลาของฟังก์ชันเอง
stats.print_stats(10)                    # แสดง 10 อันดับ

# === วิธีที่ 4: เก็บผลลัพธ์เป็น string ===
stream = io.StringIO()
stats = pstats.Stats(profiler, stream=stream)
stats.sort_stats(SortKey.CUMULATIVE)
stats.print_stats()
profile_output = stream.getvalue()
print(profile_output[:2000])  # แสดง 2000 ตัวอักษรแรก
```

### 1.2 cProfile ด้วย Decorator และ Context Manager

```python
import cProfile
import pstats
import functools
import io
from contextlib import contextmanager

# === Decorator สำหรับ profile ฟังก์ชัน ===
def profile(sort_by="cumulative", lines=20):
    """Decorator สำหรับ profile ฟังก์ชัน"""
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            profiler = cProfile.Profile()
            profiler.enable()
            try:
                result = func(*args, **kwargs)
            finally:
                profiler.disable()
                
                stream = io.StringIO()
                stats = pstats.Stats(profiler, stream=stream)
                stats.sort_stats(sort_by)
                stats.print_stats(lines)
                
                print(f"\n{'='*60}")
                print(f"Profile: {func.__name__}")
                print('='*60)
                print(stream.getvalue())
            
            return result
        return wrapper
    return decorator

# ใช้งาน decorator
@profile(sort_by="cumulative", lines=15)
def expensive_calculation(n=1000):
    """การคำนวณที่ต้องการ profile"""
    result = 0
    for i in range(n):
        result += sum(j**2 for j in range(i))
    return result

# expensive_calculation()  # จะแสดง profiling info

# === Context Manager สำหรับ profile ===
@contextmanager
def profiling(sort_by="cumulative", lines=20):
    """Context manager สำหรับ profile block of code"""
    profiler = cProfile.Profile()
    profiler.enable()
    try:
        yield profiler
    finally:
        profiler.disable()
        stream = io.StringIO()
        stats = pstats.Stats(profiler, stream=stream)
        stats.sort_stats(sort_by)
        stats.print_stats(lines)
        print(stream.getvalue())

# ใช้งาน context manager
def main():
    with profiling(sort_by="time", lines=10):
        # โค้ดที่ต้องการ profile
        data = [i**2 for i in range(100000)]
        total = sum(data)
        filtered = [x for x in data if x % 3 == 0]
    
    print(f"Total: {total}, Filtered count: {len(filtered)}")
```

### 1.3 การอ่านผลลัพธ์ cProfile

```python
# ตัวอย่างผลลัพธ์ cProfile และการตีความ:
"""
   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
      1    0.000    0.000    1.234    1.234 script.py:1(main)
   1000    1.100    0.001    1.200    0.001 script.py:10(slow_func)
  10000    0.100    0.000    0.100    0.000 script.py:20(helper)
      1    0.034    0.034    0.034    0.034 {built-in method builtins.sum}

คำอธิบาย:
- ncalls   : จำนวนครั้งที่ถูกเรียก
- tottime  : เวลารวมของฟังก์ชันเอง (ไม่รวม sub-calls)
- percall  : tottime / ncalls
- cumtime  : เวลาสะสมรวม sub-calls ทั้งหมด
- percall  : cumtime / ncalls

วิธีอ่าน:
- ถ้า tottime สูง: ฟังก์ชันนี้เองช้า
- ถ้า cumtime สูงแต่ tottime ต่ำ: sub-function ช้า
- ถ้า ncalls สูงมาก: อาจมี loop ที่ไม่จำเป็น
"""

# === การใช้ pstats filters ===
import pstats

def analyze_profile(profile_file):
    """วิเคราะห์ profile file อย่างละเอียด"""
    stats = pstats.Stats(profile_file)
    stats.strip_dirs()
    
    print("=== Top 10 ตาม Cumulative Time ===")
    stats.sort_stats("cumulative")
    stats.print_stats(10)
    
    print("\n=== Top 10 ตาม Total Time ===")
    stats.sort_stats("time")
    stats.print_stats(10)
    
    print("\n=== Top 10 ตาม Call Count ===")
    stats.sort_stats("calls")
    stats.print_stats(10)
    
    # กรองเฉพาะไฟล์ที่สนใจ
    print("\n=== เฉพาะ functions ใน myapp ===")
    stats.sort_stats("cumulative")
    stats.print_stats("myapp")  # filter ด้วย filename pattern
    
    # Callers/Callees analysis
    print("\n=== ใครเรียก fibonacci ===")
    stats.print_callers("fibonacci")
    
    print("\n=== fibonacci เรียกใคร ===")
    stats.print_callees("fibonacci")
```

---

## 2. line_profiler — วิเคราะห์แบบบรรทัดต่อบรรทัด

```bash
# ติดตั้ง
pip install line_profiler
```

### 2.1 การใช้งาน line_profiler

```python
# สำหรับใช้ใน script ตรงๆ ต้องใช้ @profile decorator
# แต่ใน production code ให้ใช้ LineProfiler object แทน

from line_profiler import LineProfiler

# === ฟังก์ชันที่ต้องการ profile ===
def process_data(data_list):
    """ประมวลผลข้อมูล — จะ profile แต่ละบรรทัด"""
    result = []                              # บรรทัดที่ 1
    
    for item in data_list:                   # บรรทัดที่ 2: loop หลัก
        # จำลองการประมวลผล
        squared = item ** 2                  # บรรทัดที่ 3: fast
        
        filtered = [x for x in range(item)   # บรรทัดที่ 4: อาจช้า
                   if x % 2 == 0]
        
        total = sum(filtered)                # บรรทัดที่ 5: fast
        
        result.append({                      # บรรทัดที่ 6
            "original": item,
            "squared": squared,
            "sum_even": total,
            "count": len(filtered)
        })
    
    return result                            # บรรทัดที่ 7

def calculate_stats(numbers):
    """คำนวณสถิติ"""
    n = len(numbers)                        # จำนวน elements
    total = sum(numbers)                    # รวมทั้งหมด
    mean = total / n                        # ค่าเฉลี่ย
    
    # Variance: ช้ากว่าเพราะ loop 2 รอบ
    variance = sum((x - mean) ** 2 for x in numbers) / n
    
    # Sorted: ช้าสำหรับข้อมูลขนาดใหญ่
    sorted_nums = sorted(numbers)
    median = sorted_nums[n // 2]
    
    return {"mean": mean, "variance": variance, "median": median}

# === ใช้งาน LineProfiler ===
def profile_with_line_profiler():
    """Profile ด้วย LineProfiler programmatically"""
    lp = LineProfiler()
    
    # เพิ่มฟังก์ชันที่ต้องการ profile
    lp.add_function(process_data)
    lp.add_function(calculate_stats)
    
    # รัน code ที่ต้องการ measure
    lp_wrapper = lp(process_data)  # wrap ฟังก์ชัน
    
    # เรียกใช้
    data = list(range(1, 101))
    result = lp_wrapper(data)
    
    # แสดงผล
    lp.print_stats()
    
    # บันทึกผล
    with open("line_profile_results.txt", "w") as f:
        lp.print_stats(stream=f)

# === ใช้ @profile decorator (สำหรับ kernprof) ===
# บันทึกในไฟล์ต่างหาก และรันด้วย:
# kernprof -l -v script.py

# ตัวอย่างไฟล์ที่ใช้กับ kernprof:
KERNPROF_SCRIPT = '''
@profile                    # decorator นี้จะทำงานได้เมื่อรันด้วย kernprof
def slow_matrix_multiply(a, b):
    rows_a = len(a)
    cols_a = len(a[0])
    cols_b = len(b[0])
    
    result = [[0] * cols_b for _ in range(rows_a)]  # สร้าง matrix
    
    for i in range(rows_a):         # 3 nested loops = O(n³) ช้ามาก
        for j in range(cols_b):
            for k in range(cols_a):
                result[i][j] += a[i][k] * b[k][j]  # bottleneck!
    
    return result

if __name__ == "__main__":
    import random
    size = 100
    matrix_a = [[random.random() for _ in range(size)] for _ in range(size)]
    matrix_b = [[random.random() for _ in range(size)] for _ in range(size)]
    slow_matrix_multiply(matrix_a, matrix_b)
'''

print("บันทึกไฟล์ด้านบนเป็น matrix_test.py แล้วรัน:")
print("kernprof -l -v matrix_test.py")
```

---

## 3. memory_profiler — ตรวจสอบการใช้หน่วยความจำ

```bash
# ติดตั้ง
pip install memory_profiler
pip install psutil  # สำหรับ cross-platform memory info
```

### 3.1 การใช้งาน memory_profiler

```python
from memory_profiler import profile as mem_profile, memory_usage
import tracemalloc
import sys

# === @profile decorator (ต้องรันกับ python -m memory_profiler) ===
# บันทึกในไฟล์ memory_test.py:
MEMORY_PROFILE_EXAMPLE = '''
from memory_profiler import profile

@profile
def memory_heavy_function():
    """ฟังก์ชันที่ใช้หน่วยความจำมาก"""
    # สร้าง list ขนาดใหญ่
    big_list = [i * 2 for i in range(1_000_000)]   # ~8 MB
    
    # แปลงเป็น set
    big_set = set(big_list)                          # ~35 MB
    
    # Dict
    big_dict = {i: i**2 for i in range(100_000)}    # ~8 MB
    
    # ลบ list (memory freed)
    del big_list
    
    # String operations
    text = " ".join(str(i) for i in range(100_000)) # ~5 MB
    
    return len(big_set) + len(big_dict)

if __name__ == "__main__":
    memory_heavy_function()
    
# รันด้วย:
# python -m memory_profiler memory_test.py
'''

# === ใช้ memory_usage() programmatically ===
def measure_memory_usage():
    """วัดการใช้หน่วยความจำของฟังก์ชัน"""
    def function_to_profile():
        """ฟังก์ชันที่จะวัด memory"""
        data = []
        for i in range(100000):
            data.append({"id": i, "value": i * 2, "name": f"item_{i}"})
        return data
    
    # วัด memory usage
    mem_before = memory_usage()[0]  # MB ก่อนรัน
    
    usage = memory_usage(
        (function_to_profile, (), {}),  # (function, args, kwargs)
        interval=0.1,                    # วัดทุก 0.1 วินาที
        timeout=30,                      # timeout 30 วินาที
        max_usage=True                   # return maximum usage
    )
    
    mem_after = memory_usage()[0]
    
    print(f"Memory before: {mem_before:.2f} MB")
    print(f"Peak memory:   {usage:.2f} MB" if isinstance(usage, float) else f"Memory samples: {usage}")
    print(f"Memory after:  {mem_after:.2f} MB")

# === tracemalloc — Built-in Memory Tracking ===
def track_memory_with_tracemalloc():
    """ใช้ tracemalloc ติดตามการจองหน่วยความจำ"""
    tracemalloc.start()         # เริ่มติดตาม
    
    # โค้ดที่ต้องการตรวจสอบ
    data = []
    for i in range(10000):
        data.append([j**2 for j in range(100)])
    
    # ดู snapshot ปัจจุบัน
    snapshot = tracemalloc.take_snapshot()
    tracemalloc.stop()
    
    # วิเคราะห์ผล
    top_stats = snapshot.statistics("lineno")
    
    print("=== Top 10 Memory Allocations ===")
    for stat in top_stats[:10]:
        print(f"{stat}")

def compare_memory_usage():
    """เปรียบเทียบ memory usage ระหว่างสองแนวทาง"""
    tracemalloc.start()
    snapshot1 = tracemalloc.take_snapshot()
    
    # วิธีที่ 1: List (ใช้ memory มาก)
    method1_data = [i * 2 for i in range(1_000_000)]
    
    snapshot2 = tracemalloc.take_snapshot()
    
    # วิธีที่ 2: Generator (ใช้ memory น้อย)
    del method1_data
    method2_data = (i * 2 for i in range(1_000_000))
    
    snapshot3 = tracemalloc.take_snapshot()
    tracemalloc.stop()
    
    # เปรียบเทียบ
    stats12 = snapshot2.compare_to(snapshot1, "lineno")
    stats23 = snapshot3.compare_to(snapshot2, "lineno")
    
    print("=== List vs Generator Memory ===")
    print("\nหลังสร้าง List:")
    for stat in stats12[:3]:
        print(f"  {stat}")
    
    print("\nหลังเปลี่ยนเป็น Generator:")
    for stat in stats23[:3]:
        print(f"  {stat}")

# === sys.getsizeof สำหรับขนาดออบเจกต์ ===
def check_object_sizes():
    """ตรวจสอบขนาดของออบเจกต์ต่างๆ"""
    objects = {
        "int(0)":              0,
        "int(1000)":           1000,
        "float":               3.14,
        "str(empty)":          "",
        "str(10 chars)":       "0123456789",
        "list(empty)":         [],
        "list(1000 ints)":     list(range(1000)),
        "dict(empty)":         {},
        "dict(100 items)":     {i: i for i in range(100)},
        "set(100 items)":      set(range(100)),
        "tuple(100 items)":    tuple(range(100)),
    }
    
    print(f"{'Object':<25} {'Size (bytes)':>15}")
    print("-" * 42)
    for name, obj in objects.items():
        size = sys.getsizeof(obj)
        print(f"{name:<25} {size:>15,}")
```

---

## 4. functools.lru_cache และ cache

`lru_cache` เก็บผลลัพธ์ของ function calls เพื่อไม่ต้องคำนวณซ้ำ

```python
import functools
import time
from typing import Any

# === lru_cache พื้นฐาน ===
@functools.lru_cache(maxsize=128)   # เก็บผลลัพธ์ 128 อัน
def fibonacci_cached(n: int) -> int:
    """Fibonacci พร้อม cache — เร็วมาก"""
    if n <= 1:
        return n
    return fibonacci_cached(n - 1) + fibonacci_cached(n - 2)

# เปรียบเทียบความเร็ว
def fibonacci_no_cache(n: int) -> int:
    """Fibonacci ไม่มี cache — ช้ามาก"""
    if n <= 1:
        return n
    return fibonacci_no_cache(n - 1) + fibonacci_no_cache(n - 2)

# ทดสอบ
start = time.perf_counter()
result = fibonacci_cached(35)
time_cached = time.perf_counter() - start
print(f"With cache: {time_cached:.6f}s, result={result}")

# fibonacci_no_cache(35) จะใช้เวลานานมาก ไม่รัน

# === cache (Python 3.9+) — ไม่จำกัดขนาด ===
@functools.cache
def expensive_lookup(key: str) -> dict:
    """จำลองการ lookup ที่แพง"""
    time.sleep(0.1)  # จำลอง database query
    return {"key": key, "value": hash(key), "timestamp": time.time()}

# === ดู cache info ===
print("\nCache info:")
print(f"  hits:    {fibonacci_cached.cache_info().hits}")
print(f"  misses:  {fibonacci_cached.cache_info().misses}")
print(f"  maxsize: {fibonacci_cached.cache_info().maxsize}")
print(f"  currsize:{fibonacci_cached.cache_info().currsize}")

# ล้าง cache
fibonacci_cached.cache_clear()

# === lru_cache กับ methods ===
class DatabaseQuery:
    """Query ที่มี cache"""
    
    def __init__(self, db_connection):
        self.db = db_connection
        self._get_user = functools.lru_cache(maxsize=1000)(self._get_user_uncached)
    
    def _get_user_uncached(self, user_id: int) -> dict:
        """Query จริงๆ"""
        # จำลอง database query
        return {"id": user_id, "name": f"User {user_id}"}
    
    def get_user(self, user_id: int) -> dict:
        return self._get_user(user_id)
    
    def invalidate_user(self, user_id: int):
        """ล้าง cache สำหรับ user นี้"""
        # lru_cache ไม่รองรับ partial invalidation
        # ต้องล้างทั้งหมดหรือใช้ library อื่น
        self._get_user.cache_clear()

# === Custom caching decorator ===
def ttl_cache(maxsize=128, ttl=300):
    """Cache ที่มี Time-To-Live (TTL)"""
    def decorator(func):
        cache = {}
        cache_times = {}
        
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            # สร้าง cache key
            key = str(args) + str(sorted(kwargs.items()))
            
            # ตรวจสอบว่า cache ยังใช้ได้
            if key in cache:
                age = time.time() - cache_times[key]
                if age < ttl:
                    return cache[key]
                else:
                    del cache[key]
                    del cache_times[key]
            
            # ล้าง cache ถ้าเกิน maxsize
            if len(cache) >= maxsize:
                oldest_key = min(cache_times, key=cache_times.get)
                del cache[oldest_key]
                del cache_times[oldest_key]
            
            # คำนวณและเก็บ cache
            result = func(*args, **kwargs)
            cache[key] = result
            cache_times[key] = time.time()
            return result
        
        wrapper.cache_clear = lambda: cache.clear() or cache_times.clear()
        wrapper.cache_info = lambda: {
            "size": len(cache),
            "maxsize": maxsize,
            "ttl": ttl
        }
        return wrapper
    return decorator

@ttl_cache(maxsize=100, ttl=60)   # Cache 60 วินาที
def get_weather(city: str) -> dict:
    """ดึงข้อมูลอากาศ (จำลอง)"""
    print(f"Fetching weather for {city}...")
    time.sleep(0.5)  # จำลอง API call
    return {"city": city, "temp": 30, "humidity": 70}

# ทดสอบ TTL cache
weather1 = get_weather("กรุงเทพ")  # miss — fetch จริง
weather2 = get_weather("กรุงเทพ")  # hit — จาก cache
print(weather1, weather2)
```

---

## 5. Redis Caching ด้วย redis-py

```bash
# ติดตั้ง
pip install redis

# รัน Redis (ด้วย Docker)
# docker run -d -p 6379:6379 redis:alpine
```

### 5.1 Redis Cache พื้นฐาน

```python
import redis
import json
import pickle
import hashlib
import functools
import time
from typing import Any, Optional, Callable

# === เชื่อมต่อ Redis ===
# วิธีที่ 1: ตรงๆ
r = redis.Redis(
    host="localhost",
    port=6379,
    db=0,
    decode_responses=True   # ส่งคืน str แทน bytes
)

# วิธีที่ 2: ผ่าน URL
r2 = redis.from_url("redis://localhost:6379/0")

# วิธีที่ 3: พร้อม connection pooling
pool = redis.ConnectionPool(
    host="localhost",
    port=6379,
    db=0,
    max_connections=20,
    decode_responses=True
)
r3 = redis.Redis(connection_pool=pool)

# === การใช้งาน Redis พื้นฐาน ===
def basic_redis_operations():
    """การ set/get ค่าพื้นฐาน"""
    
    # SET — เก็บค่า
    r.set("greeting", "สวัสดีโลก!")
    r.set("count", 42)
    r.set("temp_data", "ข้อมูลชั่วคราว", ex=300)  # หมดอายุใน 300 วินาที
    
    # GET — อ่านค่า
    greeting = r.get("greeting")
    count = r.get("count")
    print(f"greeting: {greeting}")
    print(f"count: {count}")
    
    # EXPIRE — ตั้ง TTL
    r.expire("greeting", 3600)  # หมดอายุใน 1 ชั่วโมง
    ttl = r.ttl("greeting")     # เหลือเวลากี่วินาที
    print(f"TTL: {ttl}s")
    
    # EXISTS — ตรวจสอบ
    print(f"Exists: {r.exists('greeting')}")
    
    # DELETE
    r.delete("temp_data")
    
    # INCR/DECR — counter
    r.set("page_views", 0)
    r.incr("page_views")        # เพิ่มทีละ 1
    r.incr("page_views")
    r.incrby("page_views", 10)  # เพิ่มทีละ 10
    print(f"Page views: {r.get('page_views')}")

# === Cache Object (JSON) ===
class RedisCache:
    """Cache manager สำหรับ Redis"""
    
    def __init__(self, redis_client: redis.Redis, prefix: str = "cache:", default_ttl: int = 300):
        self.redis = redis_client
        self.prefix = prefix
        self.default_ttl = default_ttl
    
    def _make_key(self, key: str) -> str:
        """สร้าง Redis key พร้อม prefix"""
        return f"{self.prefix}{key}"
    
    def get(self, key: str) -> Optional[Any]:
        """อ่าน cache"""
        value = self.redis.get(self._make_key(key))
        if value is None:
            return None
        try:
            return json.loads(value)
        except json.JSONDecodeError:
            return value
    
    def set(self, key: str, value: Any, ttl: Optional[int] = None) -> bool:
        """บันทึก cache"""
        serialized = json.dumps(value, ensure_ascii=False, default=str)
        return self.redis.setex(
            self._make_key(key),
            ttl or self.default_ttl,
            serialized
        )
    
    def delete(self, key: str) -> int:
        """ลบ cache"""
        return self.redis.delete(self._make_key(key))
    
    def exists(self, key: str) -> bool:
        """ตรวจสอบว่า key มีอยู่"""
        return bool(self.redis.exists(self._make_key(key)))
    
    def invalidate_pattern(self, pattern: str) -> int:
        """ลบ keys ที่ตรงกับ pattern"""
        full_pattern = self._make_key(pattern)
        keys = self.redis.keys(full_pattern)
        if keys:
            return self.redis.delete(*keys)
        return 0
    
    def get_or_set(self, key: str, fallback: Callable, ttl: Optional[int] = None) -> Any:
        """Get from cache หรือคำนวณและเก็บ"""
        cached = self.get(key)
        if cached is not None:
            return cached
        
        value = fallback()
        self.set(key, value, ttl)
        return value

# === Cache Decorator สำหรับ Redis ===
def redis_cache(
    redis_client: redis.Redis,
    ttl: int = 300,
    prefix: str = "func_cache:",
    key_builder: Optional[Callable] = None
):
    """Decorator สำหรับ cache function results ใน Redis"""
    
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            # สร้าง cache key
            if key_builder:
                cache_key = key_builder(*args, **kwargs)
            else:
                key_data = f"{func.__module__}.{func.__name__}:{args}:{sorted(kwargs.items())}"
                cache_key = f"{prefix}{hashlib.md5(key_data.encode()).hexdigest()}"
            
            # ลองอ่าน cache
            cached = redis_client.get(cache_key)
            if cached:
                return json.loads(cached)
            
            # คำนวณและเก็บ cache
            result = func(*args, **kwargs)
            redis_client.setex(
                cache_key,
                ttl,
                json.dumps(result, ensure_ascii=False, default=str)
            )
            return result
        
        # เพิ่ม method สำหรับล้าง cache
        def invalidate(*args, **kwargs):
            if key_builder:
                cache_key = key_builder(*args, **kwargs)
            else:
                key_data = f"{func.__module__}.{func.__name__}:{args}:{sorted(kwargs.items())}"
                cache_key = f"{prefix}{hashlib.md5(key_data.encode()).hexdigest()}"
            redis_client.delete(cache_key)
        
        wrapper.invalidate = invalidate
        return wrapper
    return decorator

# ตัวอย่างการใช้งาน
cache = RedisCache(r, prefix="myapp:", default_ttl=600)

@redis_cache(r, ttl=3600, prefix="users:")
def get_user_profile(user_id: int) -> dict:
    """ดึงข้อมูล user จาก database (จำลอง)"""
    print(f"Query database for user {user_id}...")
    time.sleep(0.2)  # จำลอง DB query
    return {
        "id": user_id,
        "name": f"User {user_id}",
        "email": f"user{user_id}@example.com",
        "created_at": "2024-01-01"
    }
```

### 5.2 Redis สำหรับ Rate Limiting และ Session

```python
import redis
import time

r = redis.Redis(host="localhost", port=6379, db=0, decode_responses=True)

# === Rate Limiting ===
def check_rate_limit(user_id: str, limit: int = 100, window: int = 3600) -> dict:
    """
    ตรวจสอบ rate limit
    limit: จำนวน requests สูงสุดใน window
    window: ช่วงเวลา (วินาที)
    """
    key = f"rate_limit:{user_id}"
    
    pipe = r.pipeline()
    now = time.time()
    window_start = now - window
    
    # ลบ requests เก่าออก
    pipe.zremrangebyscore(key, 0, window_start)
    # นับ requests ปัจจุบัน
    pipe.zcard(key)
    # เพิ่ม request ปัจจุบัน
    pipe.zadd(key, {str(now): now})
    # ตั้ง expiry
    pipe.expire(key, window)
    
    results = pipe.execute()
    current_count = results[1]
    
    return {
        "allowed": current_count < limit,
        "count": current_count,
        "limit": limit,
        "remaining": max(0, limit - current_count - 1),
        "reset_at": int(now) + window
    }

# ทดสอบ rate limiting
for i in range(5):
    result = check_rate_limit("user123", limit=3, window=60)
    print(f"Request {i+1}: allowed={result['allowed']}, remaining={result['remaining']}")

# === Distributed Lock ===
def acquire_lock(lock_name: str, ttl: int = 30) -> Optional[str]:
    """
    ขอ distributed lock
    คืน lock_id ถ้าได้ lock, None ถ้าไม่ได้
    """
    import uuid
    lock_id = str(uuid.uuid4())
    lock_key = f"lock:{lock_name}"
    
    # SET NX (set if not exists) + EX (expire)
    acquired = r.set(lock_key, lock_id, nx=True, ex=ttl)
    
    if acquired:
        return lock_id
    return None

def release_lock(lock_name: str, lock_id: str) -> bool:
    """ปล่อย lock (เฉพาะ owner เท่านั้น)"""
    lock_key = f"lock:{lock_name}"
    
    # ใช้ Lua script เพื่อ atomic check-and-delete
    lua_script = """
    if redis.call("GET", KEYS[1]) == ARGV[1] then
        return redis.call("DEL", KEYS[1])
    else
        return 0
    end
    """
    result = r.eval(lua_script, 1, lock_key, lock_id)
    return bool(result)

# ทดสอบ distributed lock
lock_id = acquire_lock("process_payment")
if lock_id:
    try:
        print("ได้ lock แล้ว กำลังประมวลผล...")
        time.sleep(1)  # จำลองงาน
    finally:
        released = release_lock("process_payment", lock_id)
        print(f"ปล่อย lock: {released}")
else:
    print("ไม่ได้ lock — มีกระบวนการอื่นทำงานอยู่")
```

---

## 6. Lazy Evaluation และ Generators vs Lists

### 6.1 Generator vs List

```python
import sys
import time

# === เปรียบเทียบหน่วยความจำ ===
def compare_memory():
    """เปรียบเทียบ memory ระหว่าง list กับ generator"""
    
    n = 1_000_000
    
    # List: สร้างข้อมูลทั้งหมดทันที
    list_data = [i ** 2 for i in range(n)]
    list_size = sys.getsizeof(list_data)
    
    # Generator: สร้างข้อมูลทีละค่า
    gen_data = (i ** 2 for i in range(n))
    gen_size = sys.getsizeof(gen_data)
    
    print(f"List size:      {list_size:>12,} bytes ({list_size/1024/1024:.2f} MB)")
    print(f"Generator size: {gen_size:>12,} bytes ({gen_size/1024:.2f} KB)")
    print(f"Ratio:          {list_size/gen_size:>12.0f}x")

compare_memory()

# === Generator Functions ===
def infinite_counter(start=0, step=1):
    """Counter ไม่มีขอบเขต"""
    current = start
    while True:
        yield current
        current += step

def fibonacci_gen():
    """Fibonacci generator"""
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

def read_large_file(filepath: str):
    """อ่านไฟล์ขนาดใหญ่ทีละบรรทัด"""
    with open(filepath, "r", encoding="utf-8") as f:
        for line in f:
            yield line.rstrip("\n")

def process_in_chunks(iterable, chunk_size=1000):
    """แบ่งข้อมูลเป็น chunks"""
    chunk = []
    for item in iterable:
        chunk.append(item)
        if len(chunk) >= chunk_size:
            yield chunk
            chunk = []
    if chunk:  # chunk สุดท้ายที่อาจไม่เต็ม
        yield chunk

# ตัวอย่างการใช้งาน
from itertools import islice, takewhile, dropwhile

# ใช้ 10 ค่าแรกจาก fibonacci
fib = fibonacci_gen()
first_10 = list(islice(fib, 10))
print(f"Fibonacci 10 ค่าแรก: {first_10}")

# ใช้ fibonacci จนกว่าจะเกิน 100
fib = fibonacci_gen()
under_100 = list(takewhile(lambda x: x <= 100, fib))
print(f"Fibonacci ≤ 100: {under_100}")

# === Generator Pipelines ===
def data_pipeline():
    """ตัวอย่าง pipeline ด้วย generators"""
    
    # ข้อมูล input
    raw_data = range(1, 10001)
    
    # Step 1: Filter — กรองเฉพาะเลขคู่
    step1 = (x for x in raw_data if x % 2 == 0)
    
    # Step 2: Transform — คำนวณ square root
    import math
    step2 = (math.sqrt(x) for x in step1)
    
    # Step 3: Filter — กรองเฉพาะที่เป็นจำนวนเต็ม
    step3 = (x for x in step2 if x.is_integer())
    
    # Step 4: Format
    step4 = (f"{int(x):04d}" for x in step3)
    
    # เรียกใช้ทั้ง pipeline
    results = list(step4)
    print(f"Pipeline results count: {len(results)}")
    print(f"First 5: {results[:5]}")

data_pipeline()
```

### 6.2 Lazy Evaluation Patterns

```python
from typing import Iterator, Callable, TypeVar, Generic
import functools

T = TypeVar("T")

# === Lazy Property ===
class LazyProperty:
    """Descriptor สำหรับ lazy computation"""
    
    def __init__(self, func: Callable):
        self.func = func
        self.attrname = None
    
    def __set_name__(self, owner, name):
        self.attrname = name
    
    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        # คำนวณเฉพาะครั้งแรก แล้วเก็บไว้
        value = self.func(obj)
        setattr(obj, self.attrname, value)  # แทนที่ descriptor ด้วยค่าจริง
        return value

class DataProcessor:
    """ตัวอย่างคลาสที่ใช้ Lazy Properties"""
    
    def __init__(self, data: list):
        self.data = data
    
    @LazyProperty
    def sorted_data(self):
        """เรียงข้อมูล — คำนวณเฉพาะเมื่อต้องการ"""
        print("Computing sorted_data...")
        return sorted(self.data)
    
    @LazyProperty
    def statistics(self):
        """คำนวณสถิติ — คำนวณเฉพาะเมื่อต้องการ"""
        print("Computing statistics...")
        n = len(self.data)
        total = sum(self.data)
        mean = total / n
        return {
            "count": n,
            "sum": total,
            "mean": mean,
            "min": min(self.data),
            "max": max(self.data)
        }

# ทดสอบ
processor = DataProcessor([3, 1, 4, 1, 5, 9, 2, 6, 5, 3])
print("สร้าง DataProcessor แล้ว (ยังไม่คำนวณ)")
print(f"Sorted: {processor.sorted_data}")  # คำนวณตอนนี้
print(f"Sorted again: {processor.sorted_data}")  # ใช้ cache
print(f"Stats: {processor.statistics}")

# === Lazy Loading Class ===
class LazyLoader:
    """โหลดข้อมูลเฉพาะเมื่อต้องการ"""
    
    def __init__(self, loader_func: Callable):
        self._loader = loader_func
        self._data = None
        self._loaded = False
    
    def __getattr__(self, name):
        if not self._loaded:
            print(f"Loading data (triggered by .{name})...")
            self._data = self._loader()
            self._loaded = True
        return getattr(self._data, name)

def load_config():
    """จำลองการโหลด config ที่ช้า"""
    import time
    time.sleep(0.5)  # จำลองการอ่านไฟล์
    return {
        "database": {"host": "localhost", "port": 5432},
        "redis": {"host": "localhost", "port": 6379},
        "debug": True
    }

# สร้าง lazy config — ยังไม่โหลด
config = LazyLoader(load_config)
print("สร้าง config object แล้ว (ยังไม่โหลด)")

# โหลดจริงเมื่อเข้าถึง
# db_config = config["database"]  # trigger loading
```

---

## 7. หลีกเลี่ยง Performance Pitfalls

### 7.1 String Concatenation

```python
import time

n = 100000

# === แบบผิด: String concatenation ใน loop ===
def slow_string_build():
    """O(n²) — แต่ละครั้งสร้าง string ใหม่"""
    result = ""
    for i in range(n):
        result += str(i) + ","   # สร้าง string ใหม่ทุกครั้ง!
    return result

# === แบบถูก: join() ===
def fast_string_build():
    """O(n) — รวมครั้งเดียว"""
    parts = []
    for i in range(n):
        parts.append(str(i))
    return ",".join(parts)

# หรือ comprehension
def fastest_string_build():
    return ",".join(str(i) for i in range(n))

# เปรียบเทียบ
start = time.perf_counter()
slow_string_build()
slow_time = time.perf_counter() - start

start = time.perf_counter()
fast_string_build()
fast_time = time.perf_counter() - start

print(f"Slow (+=):  {slow_time:.4f}s")
print(f"Fast (join):{fast_time:.4f}s")
print(f"Speedup:    {slow_time/fast_time:.1f}x")
```

### 7.2 การค้นหาใน List vs Set

```python
import random
import time

data = list(range(1000000))
data_set = set(data)
lookup_values = [random.randint(0, 2000000) for _ in range(1000)]

# === แบบผิด: ค้นหาใน list — O(n) ===
start = time.perf_counter()
results_list = [v in data for v in lookup_values]      # O(n) ต่อการค้นหา
list_time = time.perf_counter() - start

# === แบบถูก: ค้นหาใน set — O(1) ===
start = time.perf_counter()
results_set = [v in data_set for v in lookup_values]   # O(1) ต่อการค้นหา
set_time = time.perf_counter() - start

print(f"List lookup: {list_time:.4f}s")
print(f"Set lookup:  {set_time:.6f}s")
print(f"Speedup:     {list_time/set_time:.0f}x")
```

### 7.3 Loop Optimizations

```python
import time

# === แบบผิด: การ lookup ซ้ำใน loop ===
def slow_loop(data):
    result = []
    for i in range(len(data)):              # len() ถูกเรียกทุกรอบ
        result.append(data[i] * 2)         # attribute lookup ทุกรอบ
    return result

# === แบบดีขึ้น: ลด lookups ===
def better_loop(data):
    result = []
    append = result.append                  # cache method
    n = len(data)                          # cache length
    for i in range(n):
        append(data[i] * 2)
    return result

# === แบบที่ดีที่สุด: List comprehension ===
def best_loop(data):
    return [x * 2 for x in data]           # ใช้ list comprehension

# === หลีกเลี่ยง Global Variable ใน Loop ===
import math

# ช้ากว่า: global lookup ทุกรอบ
def slow_math(n):
    results = []
    for i in range(n):
        results.append(math.sqrt(i))       # lookup math.sqrt ทุกรอบ
    return results

# เร็วกว่า: local reference
def fast_math(n):
    sqrt = math.sqrt                       # cache เป็น local variable
    results = []
    for i in range(n):
        results.append(sqrt(i))            # local lookup เร็วกว่า
    return results

# === map() แทน loop ===
def using_map(n):
    return list(map(math.sqrt, range(n)))  # map ใช้ C-level loop

# === หลีกเลี่ยงการสร้าง intermediate list ===
# แบบผิด: สร้าง list ขนาดใหญ่ชั่วคราว
def slow_filter_sum(data):
    filtered = [x for x in data if x % 2 == 0]   # สร้าง list ชั่วคราว
    return sum(filtered)

# แบบถูก: ใช้ generator expression
def fast_filter_sum(data):
    return sum(x for x in data if x % 2 == 0)    # ไม่สร้าง list ชั่วคราว
```

### 7.4 Algorithm Complexity

```python
# === O(n²) vs O(n log n) ===
import random

def bubble_sort(arr):
    """O(n²) — ช้าสำหรับข้อมูลใหญ่"""
    n = len(arr)
    arr = arr.copy()
    for i in range(n):
        for j in range(0, n - i - 1):
            if arr[j] > arr[j + 1]:
                arr[j], arr[j + 1] = arr[j + 1], arr[j]
    return arr

def python_sort(arr):
    """O(n log n) — Timsort built-in"""
    return sorted(arr)

# === N+1 Query Problem (Django/SQLAlchemy) ===
# แบบผิด: N+1 queries
def get_users_with_posts_bad(db_session):
    """ทำ N+1 queries — 1 query สำหรับ users + N queries สำหรับ posts"""
    users = db_session.query(User).all()         # 1 query
    result = []
    for user in users:
        # !! N queries — 1 ต่อ user !!
        posts = db_session.query(Post).filter_by(user_id=user.id).all()
        result.append({"user": user.name, "post_count": len(posts)})
    return result

# แบบถูก: Eager loading
def get_users_with_posts_good(db_session):
    """1 query ด้วย JOIN"""
    from sqlalchemy.orm import joinedload
    users = db_session.query(User).options(
        joinedload(User.posts)              # JOIN เดียว
    ).all()
    return [
        {"user": u.name, "post_count": len(u.posts)}
        for u in users
    ]

# === หลีกเลี่ยง Repeated Dict/List Creation ===
# แบบผิด: สร้าง dict ใหม่ทุกรอบ
def create_config_bad(items):
    configs = []
    for item in items:
        configs.append({              # สร้าง dict ใหม่ทุกครั้ง
            "id": item.id,
            "name": item.name,
            "enabled": True,
            "timeout": 30,
            "retry": 3,
        })
    return configs

# แบบดีกว่า: ใช้ template หรือ dataclass
from dataclasses import dataclass

@dataclass
class Config:
    id: int
    name: str
    enabled: bool = True
    timeout: int = 30
    retry: int = 3

def create_config_good(items):
    return [Config(id=item.id, name=item.name) for item in items]
```

---

## 8. Benchmarking ด้วย timeit

```python
import timeit
import time
from typing import Callable

# === timeit.timeit() — วิธีที่ง่ายสุด ===
def benchmark_basic():
    """Benchmark แบบง่าย"""
    
    # วัดเวลา statement
    time_list = timeit.timeit(
        stmt="[i**2 for i in range(1000)]",  # โค้ดที่วัด
        number=10000                           # รันกี่ครั้ง
    )
    time_gen = timeit.timeit(
        stmt="list(i**2 for i in range(1000))",
        number=10000
    )
    
    print(f"List comprehension: {time_list:.4f}s")
    print(f"Generator to list:  {time_gen:.4f}s")
    print(f"Winner: {'LC' if time_list < time_gen else 'Gen'} ({min(time_list, time_gen):.4f}s)")

# === timeit.repeat() — รันหลายรอบเพื่อความแม่นยำ ===
def benchmark_repeat():
    """Benchmark ที่น่าเชื่อถือกว่า"""
    
    setups = [
        ("dict.get", "d = {i:i for i in range(100)}", "d.get(50, None)"),
        ("dict[]",   "d = {i:i for i in range(100)}", "d[50] if 50 in d else None"),
    ]
    
    for name, setup, stmt in setups:
        times = timeit.repeat(
            stmt=stmt,
            setup=setup,
            number=100000,    # 100k รันต่อรอบ
            repeat=5          # 5 รอบ
        )
        min_time = min(times)
        avg_time = sum(times) / len(times)
        print(f"{name:<20}: min={min_time:.4f}s avg={avg_time:.4f}s")

# === Custom Benchmark Framework ===
class Benchmark:
    """Framework สำหรับ benchmarking"""
    
    def __init__(self, name: str = "Benchmark"):
        self.name = name
        self.results: dict[str, list[float]] = {}
    
    def run(
        self,
        label: str,
        func: Callable,
        *args,
        n: int = 100,
        warmup: int = 10,
        **kwargs
    ) -> float:
        """รัน benchmark"""
        # Warmup
        for _ in range(warmup):
            func(*args, **kwargs)
        
        # Measure
        times = []
        for _ in range(n):
            start = time.perf_counter()
            func(*args, **kwargs)
            elapsed = time.perf_counter() - start
            times.append(elapsed)
        
        self.results[label] = times
        return min(times)
    
    def report(self):
        """แสดงผล benchmark"""
        print(f"\n{'='*60}")
        print(f"Benchmark: {self.name}")
        print(f"{'='*60}")
        print(f"{'Label':<30} {'Min':>10} {'Avg':>10} {'Max':>10}")
        print(f"{'-'*60}")
        
        baseline = None
        for label, times in self.results.items():
            min_t = min(times)
            avg_t = sum(times) / len(times)
            max_t = max(times)
            
            if baseline is None:
                baseline = min_t
                speedup = ""
            else:
                speedup = f" ({baseline/min_t:.1f}x)"
            
            print(f"{label:<30} {min_t*1000:>9.3f}ms {avg_t*1000:>9.3f}ms {max_t*1000:>9.3f}ms{speedup}")

# === ตัวอย่างการใช้งาน Benchmark ===
def demo_benchmark():
    """เปรียบเทียบ string building methods"""
    
    bench = Benchmark("String Building")
    
    n = 1000
    
    # วิธีที่ 1: +=
    def method_concat():
        s = ""
        for i in range(n):
            s += str(i)
        return s
    
    # วิธีที่ 2: join
    def method_join():
        return "".join(str(i) for i in range(n))
    
    # วิธีที่ 3: io.StringIO
    import io
    def method_stringio():
        buf = io.StringIO()
        for i in range(n):
            buf.write(str(i))
        return buf.getvalue()
    
    # วิธีที่ 4: f-string ใน list
    def method_fstring():
        return "".join(f"{i}" for i in range(n))
    
    bench.run("String +=",   method_concat, n=500)
    bench.run("''.join()",   method_join, n=500)
    bench.run("StringIO",    method_stringio, n=500)
    bench.run("f-string join", method_fstring, n=500)
    
    bench.report()

demo_benchmark()
benchmark_basic()
benchmark_repeat()
```

---

## 9. เทคนิคเพิ่มเติม

### 9.1 NumPy สำหรับ Numerical Operations

```python
# ถ้ามีการคำนวณตัวเลขเยอะๆ NumPy เร็วกว่า Python loops มาก
import time

# จำลองการคำนวณโดยไม่ต้อง import จริง
def python_vector_add(a, b):
    """Python loop"""
    return [x + y for x, y in zip(a, b)]

# NumPy version เร็วกว่า ~100x สำหรับ array ขนาดใหญ่:
# import numpy as np
# def numpy_vector_add(a, b):
#     return np.array(a) + np.array(b)  # vectorized!

# === Slots สำหรับ Memory Optimization ===
import sys

class WithSlots:
    """Class ที่ใช้ __slots__ ประหยัด memory"""
    __slots__ = ["x", "y", "name"]  # กำหนด attributes ล่วงหน้า
    
    def __init__(self, x, y, name):
        self.x = x
        self.y = y
        self.name = name

class WithoutSlots:
    """Class ปกติที่มี __dict__"""
    
    def __init__(self, x, y, name):
        self.x = x
        self.y = y
        self.name = name

with_slots = WithSlots(1, 2, "test")
without_slots = WithoutSlots(1, 2, "test")

print(f"With slots:    {sys.getsizeof(with_slots)} bytes")
print(f"Without slots: {sys.getsizeof(without_slots)} bytes")
# __slots__ ลดขนาดได้ ~40-50%
```

### 9.2 Concurrent Execution

```python
import concurrent.futures
import time

def cpu_bound_task(n: int) -> int:
    """งานที่ใช้ CPU มาก"""
    total = 0
    for i in range(n):
        total += i ** 2
    return total

def io_bound_task(url: str) -> str:
    """งานที่รอ I/O"""
    time.sleep(0.1)  # จำลอง network request
    return f"Result from {url}"

# === ThreadPoolExecutor สำหรับ I/O bound ===
def parallel_io_bound():
    urls = [f"https://api.example.com/{i}" for i in range(10)]
    
    with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
        # submit ทุกงานพร้อมกัน
        futures = {executor.submit(io_bound_task, url): url for url in urls}
        
        results = []
        for future in concurrent.futures.as_completed(futures):
            url = futures[future]
            try:
                result = future.result()
                results.append(result)
            except Exception as e:
                print(f"Error for {url}: {e}")
    
    return results

# === ProcessPoolExecutor สำหรับ CPU bound ===
def parallel_cpu_bound():
    numbers = [10**6] * 8  # 8 งาน
    
    with concurrent.futures.ProcessPoolExecutor() as executor:
        results = list(executor.map(cpu_bound_task, numbers))
    
    return results

# ทดสอบ
start = time.perf_counter()
results = parallel_io_bound()
elapsed = time.perf_counter() - start
print(f"Parallel I/O: {len(results)} results in {elapsed:.2f}s")
```

---

## 10. สรุป Part 048

✅ **cProfile + pstats** — Profiling เพื่อหา bottleneck โดยวัดเวลาแต่ละ function

✅ **line_profiler** — วิเคราะห์แบบบรรทัดต่อบรรทัด เพื่อหาบรรทัดที่ช้า

✅ **memory_profiler + tracemalloc** — ตรวจสอบการใช้หน่วยความจำ

✅ **lru_cache + cache** — Cache ผลลัพธ์ function เพื่อไม่คำนวณซ้ำ

✅ **Redis caching** — Distributed cache สำหรับ production ด้วย redis-py

✅ **Generators** — ประหยัด memory ด้วย lazy evaluation

✅ **Common Pitfalls** — String concatenation, List vs Set lookup, N+1 queries

✅ **timeit** — Benchmark code อย่างถูกต้องและน่าเชื่อถือ

## ➡️ ถัดไป: Part 049 - Data Validation with Pydantic v2

*Part 048/100+ | Python Course - Beginner to World-Class*
