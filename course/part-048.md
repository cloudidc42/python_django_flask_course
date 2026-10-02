# Part 048: Performance Optimization
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- Profile code ด้วย cProfile และ line_profiler
- Memory profiling ด้วย memory_profiler
- Cache ด้วย functools.lru_cache
- Redis caching strategies
- Lazy evaluation และ generators
- Algorithm optimization techniques
- Concurrency optimization

---

## 1. Profiling ด้วย cProfile

```python
import cProfile
import pstats
import io
import time
from functools import wraps

# === cProfile Usage ===
def slow_function():
    """Function ที่ช้า (จำลอง)"""
    result = []
    for i in range(10000):
        # String concatenation ใน loop (ช้า)
        s = ""
        for j in range(100):
            s += str(j)
        result.append(s)
    return result


def fast_function():
    """Function ที่เร็วกว่า"""
    return ["".join(str(j) for j in range(100)) for i in range(10000)]


# === Profile ด้วย cProfile ===
def profile_function():
    # Method 1: context manager
    profiler = cProfile.Profile()
    profiler.enable()
    
    slow_function()
    
    profiler.disable()
    
    # แสดงผล
    stream = io.StringIO()
    stats = pstats.Stats(profiler, stream=stream)
    stats.sort_stats("cumulative")  # เรียงตาม cumulative time
    stats.print_stats(20)  # แสดง 20 อันดับแรก
    print(stream.getvalue())


# Method 2: decorator
def profile(func):
    """Profile decorator"""
    @wraps(func)
    def wrapper(*args, **kwargs):
        profiler = cProfile.Profile()
        profiler.enable()
        
        result = func(*args, **kwargs)
        
        profiler.disable()
        
        stream = io.StringIO()
        stats = pstats.Stats(profiler, stream=stream)
        stats.sort_stats("cumulative")
        stats.print_stats(10)
        
        print(f"\n=== Profile: {func.__name__} ===")
        print(stream.getvalue())
        
        return result
    return wrapper


@profile
def compute_primes(n: int) -> list:
    """หาเลขจำนวนเฉพาะด้วย Sieve of Eratosthenes"""
    if n < 2:
        return []
    
    sieve = [True] * (n + 1)
    sieve[0] = sieve[1] = False
    
    for i in range(2, int(n ** 0.5) + 1):
        if sieve[i]:
            for j in range(i * i, n + 1, i):
                sieve[j] = False
    
    return [i for i in range(2, n + 1) if sieve[i]]


# compute_primes(100000)  # Uncomment เพื่อ profile


# === timeit สำหรับ Micro-benchmarks ===
import timeit

def benchmark_comparison():
    """เปรียบเทียบความเร็วของวิธีต่างๆ"""
    
    # Test 1: List vs Generator
    n = 10000
    
    list_time = timeit.timeit(
        lambda: list(range(n)),
        number=1000
    )
    
    gen_time = timeit.timeit(
        lambda: sum(range(n)),
        number=1000
    )
    
    print(f"List comprehension: {list_time:.4f}s")
    print(f"Generator sum: {gen_time:.4f}s")
    
    # Test 2: String concatenation
    str_concat = timeit.timeit(
        'result = ""; [result := result + str(i) for i in range(100)]',
        number=1000
    )
    
    str_join = timeit.timeit(
        '"".join(str(i) for i in range(100))',
        number=1000
    )
    
    print(f"\nString concat: {str_concat:.4f}s")
    print(f"String join: {str_join:.4f}s")
    print(f"Join is {str_concat/str_join:.1f}x faster")
    
    # Test 3: Dict vs defaultdict
    regular_dict = timeit.timeit("""
d = {}
for i in range(1000):
    key = i % 100
    if key not in d:
        d[key] = 0
    d[key] += 1
""", number=100)
    
    default_dict = timeit.timeit("""
from collections import defaultdict
d = defaultdict(int)
for i in range(1000):
    d[i % 100] += 1
""", number=100)
    
    print(f"\nRegular dict: {regular_dict:.4f}s")
    print(f"defaultdict: {default_dict:.4f}s")


benchmark_comparison()
```

---

## 2. line_profiler

```bash
pip install line_profiler
```

```python
# @profile decorator จาก line_profiler
# รัน: kernprof -l -v script.py

# หรือใช้ inline
from line_profiler import LineProfiler

def expensive_computation(data: list) -> list:
    """Function ที่ต้องการ line-by-line profiling"""
    results = []
    
    for item in data:
        # Line 1: คูณ
        doubled = item * 2
        
        # Line 2: เงื่อนไข
        if doubled > 100:
            # Line 3: append เฉพาะเมื่อเงื่อนไขเป็นจริง
            results.append(doubled)
        else:
            # Line 4: ทำ transformation อื่น
            results.append(doubled + 10)
    
    return results


def profile_line_by_line():
    """Profile function ทีละบรรทัด"""
    profiler = LineProfiler()
    profiler.add_function(expensive_computation)
    
    data = list(range(10000))
    profiler.runcall(expensive_computation, data)
    
    profiler.print_stats()


profile_line_by_line()
```

---

## 3. Memory Profiling

```bash
pip install memory_profiler psutil
```

```python
import psutil
import os
import tracemalloc
from typing import Iterator

# === tracemalloc (Built-in) ===
def demo_tracemalloc():
    """ตรวจสอบ memory allocation"""
    
    tracemalloc.start()
    
    # Code ที่ต้องการ track
    data = []
    for i in range(100000):
        data.append({"id": i, "value": i * 2, "name": f"item_{i}"})
    
    # ดู snapshot
    snapshot = tracemalloc.take_snapshot()
    top_stats = snapshot.statistics("lineno")
    
    print("Memory allocation top 3:")
    for stat in top_stats[:3]:
        print(f"  {stat}")
    
    # ดู current memory
    current, peak = tracemalloc.get_traced_memory()
    print(f"Current memory: {current / 1024 / 1024:.2f} MB")
    print(f"Peak memory: {peak / 1024 / 1024:.2f} MB")
    
    tracemalloc.stop()


# === psutil Memory Monitoring ===
def get_memory_usage() -> float:
    """ดู memory usage ของ process ปัจจุบัน (MB)"""
    process = psutil.Process(os.getpid())
    return process.memory_info().rss / 1024 / 1024  # Convert to MB


def memory_benchmark(func, *args):
    """วัด memory ก่อนและหลัง function"""
    before = get_memory_usage()
    result = func(*args)
    after = get_memory_usage()
    
    print(f"{func.__name__}: {after - before:.2f} MB used")
    return result


# === เปรียบเทียบ List vs Generator Memory ===
def create_list(n: int) -> list:
    """สร้าง list ของตัวเลข - เก็บทั้งหมดใน memory"""
    return [i ** 2 for i in range(n)]


def create_generator(n: int) -> Iterator[int]:
    """สร้าง generator - ไม่เก็บทั้งหมดใน memory"""
    for i in range(n):
        yield i ** 2


def demo_memory_comparison():
    n = 1_000_000
    
    print("=== Memory Comparison ===")
    
    # List
    import sys
    lst = [i ** 2 for i in range(1000)]
    gen = (i ** 2 for i in range(1000))
    
    print(f"List size (1000 elements): {sys.getsizeof(lst)} bytes")
    print(f"Generator size: {sys.getsizeof(gen)} bytes")
    
    # Large scale
    before = get_memory_usage()
    big_list = create_list(100000)
    after_list = get_memory_usage()
    
    print(f"\nList (100k): {after_list - before:.2f} MB")
    
    del big_list
    
    before = get_memory_usage()
    # Generator ไม่ใช้ memory จนกว่าจะ iterate
    gen = create_generator(100000)
    after_gen = get_memory_usage()
    
    print(f"Generator (100k): {after_gen - before:.4f} MB")
    
    # Process all items จาก generator
    total = sum(gen)
    print(f"Generator sum: {total}")


demo_tracemalloc()
demo_memory_comparison()
```

---

## 4. functools.lru_cache

```python
from functools import lru_cache, cache
import time
from typing import Callable

# === lru_cache ===
@lru_cache(maxsize=128)  # Cache ผลลัพธ์สูงสุด 128 entries
def fibonacci(n: int) -> int:
    """Fibonacci ที่ cache ผลลัพธ์"""
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)


# @cache เหมือน @lru_cache(maxsize=None) - cache ไม่จำกัด
@cache
def compute_factorial(n: int) -> int:
    if n <= 1:
        return 1
    return n * compute_factorial(n - 1)


def demo_cache():
    # ไม่มี cache
    def fib_no_cache(n: int) -> int:
        if n < 2:
            return n
        return fib_no_cache(n - 1) + fib_no_cache(n - 2)
    
    # วัดเวลา
    n = 35
    
    start = time.time()
    result = fib_no_cache(n)
    no_cache_time = time.time() - start
    
    start = time.time()
    result = fibonacci(n)
    cached_time = time.time() - start
    
    print(f"fib({n}) without cache: {no_cache_time:.4f}s")
    print(f"fib({n}) with cache: {cached_time:.4f}s")
    print(f"Speedup: {no_cache_time / cached_time:.0f}x faster")
    
    # ดู cache info
    info = fibonacci.cache_info()
    print(f"\nCache info: hits={info.hits}, misses={info.misses}")
    
    # Clear cache
    fibonacci.cache_clear()
    print(f"After clear: {fibonacci.cache_info()}")


demo_cache()


# === Cache สำหรับ Method (Instance) ===
from functools import cached_property

class Circle:
    def __init__(self, radius: float):
        self.radius = radius
    
    @cached_property
    def area(self) -> float:
        """คำนวณ area เพียงครั้งเดียว แล้ว cache ไว้"""
        print("Computing area...")  # จะแสดงครั้งเดียว
        import math
        return math.pi * self.radius ** 2
    
    @cached_property
    def circumference(self) -> float:
        """Circumference - cached"""
        import math
        return 2 * math.pi * self.radius


c = Circle(5)
print(f"Area: {c.area:.2f}")   # Computing...
print(f"Area: {c.area:.2f}")   # จาก cache ไม่ compute ใหม่
print(f"Circumference: {c.circumference:.2f}")


# === TTL Cache (Expire after time) ===
import time
from dataclasses import dataclass, field
from typing import Any, Optional

@dataclass
class CacheEntry:
    value: Any
    expires_at: float

class TTLCache:
    """Cache ที่หมดอายุหลังเวลาที่กำหนด"""
    
    def __init__(self, ttl_seconds: int = 300, max_size: int = 1000):
        self.ttl = ttl_seconds
        self.max_size = max_size
        self._cache: dict[str, CacheEntry] = {}
    
    def get(self, key: str) -> Optional[Any]:
        entry = self._cache.get(key)
        if entry is None:
            return None
        
        if time.time() > entry.expires_at:
            del self._cache[key]
            return None
        
        return entry.value
    
    def set(self, key: str, value: Any):
        # Evict ถ้า cache เต็ม
        if len(self._cache) >= self.max_size:
            self._evict()
        
        self._cache[key] = CacheEntry(
            value=value,
            expires_at=time.time() + self.ttl
        )
    
    def _evict(self):
        """ลบ entries ที่หมดอายุ"""
        now = time.time()
        expired = [k for k, v in self._cache.items() if v.expires_at < now]
        for key in expired:
            del self._cache[key]
        
        # ถ้ายังเต็ม ลบ entry เก่าสุด
        if len(self._cache) >= self.max_size:
            oldest = min(self._cache, key=lambda k: self._cache[k].expires_at)
            del self._cache[oldest]
    
    def __len__(self) -> int:
        return len(self._cache)


def ttl_cache(ttl: int = 300):
    """Decorator สำหรับ TTL cache"""
    cache = TTLCache(ttl_seconds=ttl)
    
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        def wrapper(*args, **kwargs):
            key = str(args) + str(sorted(kwargs.items()))
            
            cached = cache.get(key)
            if cached is not None:
                return cached
            
            result = func(*args, **kwargs)
            cache.set(key, result)
            return result
        
        wrapper.cache = cache
        return wrapper
    
    return decorator


@ttl_cache(ttl=60)  # Cache นาน 60 วินาที
def get_user_from_db(user_id: int) -> dict:
    """Simulate DB query"""
    time.sleep(0.1)  # Simulate slow DB
    return {"id": user_id, "name": f"User{user_id}"}


# ทดสอบ TTL Cache
start = time.time()
user = get_user_from_db(1)
first_time = time.time() - start

start = time.time()
user = get_user_from_db(1)  # จาก cache
second_time = time.time() - start

print(f"First call: {first_time:.3f}s")
print(f"Second call (cached): {second_time:.3f}s")
print(f"Speedup: {first_time/second_time:.0f}x")
```

---

## 5. Redis Caching

```python
import redis
import json
import hashlib
import functools
import time
from typing import Any, Optional, Callable

# เชื่อม Redis
# redis_client = redis.Redis(host="localhost", port=6379, db=0, decode_responses=True)

# Mock Redis สำหรับ demo
class MockRedis:
    """Mock Redis สำหรับ testing"""
    def __init__(self):
        self._data = {}
        self._expires = {}
    
    def get(self, key: str) -> Optional[str]:
        if key in self._expires and time.time() > self._expires[key]:
            del self._data[key]
            del self._expires[key]
            return None
        return self._data.get(key)
    
    def setex(self, key: str, seconds: int, value: str):
        self._data[key] = value
        self._expires[key] = time.time() + seconds
    
    def delete(self, *keys):
        for key in keys:
            self._data.pop(key, None)
            self._expires.pop(key, None)
    
    def exists(self, key: str) -> bool:
        return key in self._data
    
    def keys(self, pattern: str) -> list:
        import fnmatch
        return [k for k in self._data.keys() if fnmatch.fnmatch(k, pattern)]


redis_client = MockRedis()


# === Redis Cache Decorator ===
def redis_cache(ttl: int = 300, prefix: str = "cache"):
    """
    Cache function results ใน Redis
    ttl: seconds
    prefix: key prefix
    """
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            # สร้าง cache key
            key_data = f"{func.__name__}:{args}:{sorted(kwargs.items())}"
            cache_key = f"{prefix}:{hashlib.md5(key_data.encode()).hexdigest()}"
            
            # ตรวจสอบ cache
            cached = redis_client.get(cache_key)
            if cached:
                print(f"  [Cache HIT] {func.__name__}")
                return json.loads(cached)
            
            print(f"  [Cache MISS] {func.__name__}")
            
            # Call function
            result = func(*args, **kwargs)
            
            # เก็บใน cache
            redis_client.setex(cache_key, ttl, json.dumps(result))
            
            return result
        
        def invalidate(*args, **kwargs):
            """ลบ cache entry"""
            key_data = f"{func.__name__}:{args}:{sorted(kwargs.items())}"
            cache_key = f"{prefix}:{hashlib.md5(key_data.encode()).hexdigest()}"
            redis_client.delete(cache_key)
        
        wrapper.invalidate = invalidate
        return wrapper
    
    return decorator


# === Caching Strategies ===
class UserService:
    """Service ที่ใช้ Redis Cache"""
    
    @redis_cache(ttl=300, prefix="user")
    def get_user(self, user_id: int) -> dict:
        """Simulate DB query"""
        time.sleep(0.05)  # Simulate slow DB
        return {"id": user_id, "name": f"User {user_id}", "email": f"user{user_id}@example.com"}
    
    def update_user(self, user_id: int, data: dict) -> dict:
        """อัปเดต user และ invalidate cache"""
        # Update in DB (mock)
        updated = {"id": user_id, **data}
        
        # Invalidate cache
        self.get_user.invalidate(self, user_id)
        
        return updated
    
    @redis_cache(ttl=60, prefix="user_list")
    def get_all_users(self, page: int = 1, per_page: int = 10) -> list:
        """Cache รายการ users"""
        time.sleep(0.1)  # Slow DB query
        return [{"id": i, "name": f"User {i}"} for i in range(per_page)]


# Cache Invalidation Patterns
class CacheManager:
    """จัดการ cache invalidation"""
    
    KEY_PATTERNS = {
        "user": "user:*",
        "product": "product:*",
        "order": "order:*",
    }
    
    def invalidate_pattern(self, pattern: str):
        """ลบ cache ทั้งหมดที่ match pattern"""
        keys = redis_client.keys(pattern)
        if keys:
            redis_client.delete(*keys)
            print(f"Invalidated {len(keys)} cache entries")
    
    def invalidate_user_cache(self, user_id: int = None):
        """ลบ cache ที่เกี่ยวกับ user"""
        if user_id:
            self.invalidate_pattern(f"user:*{user_id}*")
        else:
            self.invalidate_pattern("user:*")


# ทดสอบ Redis Cache
def demo_redis_cache():
    service = UserService()
    
    print("=== Redis Cache Demo ===")
    
    # First call - miss
    user = service.get_user(1)
    print(f"User: {user['name']}")
    
    # Second call - hit
    user = service.get_user(1)
    print(f"User again: {user['name']}")
    
    # Different user - miss
    user2 = service.get_user(2)
    print(f"User 2: {user2['name']}")
    
    # Get all users
    users = service.get_all_users(page=1)
    print(f"\nAll users (first call): {len(users)} items")
    users = service.get_all_users(page=1)  # From cache
    print(f"All users (cached): {len(users)} items")


demo_redis_cache()
```

---

## 6. Generator และ Lazy Evaluation

```python
from typing import Iterator, Generator
import itertools

# === Generators ประหยัด Memory ===
def read_large_file_generator(filepath: str) -> Iterator[str]:
    """อ่านไฟล์ขนาดใหญ่ทีละบรรทัด"""
    with open(filepath) as f:
        for line in f:
            yield line.strip()
    # ไม่ต้อง load ทั้งไฟล์เข้า memory


def process_large_csv() -> Generator[dict, None, None]:
    """Process CSV ขนาดใหญ่แบบ streaming"""
    import csv
    import io
    
    # Mock CSV data
    csv_data = "name,age,email\nAlice,30,alice@example.com\nBob,25,bob@example.com"
    
    reader = csv.DictReader(io.StringIO(csv_data))
    for row in reader:
        # Transform each row
        yield {
            "name": row["name"].strip(),
            "age": int(row["age"]),
            "email": row["email"].lower(),
        }


def demo_generators():
    # Process สำหรับทุกแถว โดยไม่ load ทั้งหมด
    for user in process_large_csv():
        print(f"Processing: {user['name']}")
    
    # Generator pipeline
    numbers = range(1000000)
    
    # Pipeline: filter → map → limit
    pipeline = (
        x * 2                           # map
        for x in numbers                # source
        if x % 2 == 0                   # filter
    )
    
    # ดึงเฉพาะที่ต้องการ
    first_10 = list(itertools.islice(pipeline, 10))
    print(f"First 10 even*2: {first_10}")
    
    # Memory usage จาก sys
    import sys
    lst = list(range(1000000))
    gen = range(1000000)
    
    print(f"List size: {sys.getsizeof(lst) / 1024:.0f} KB")
    print(f"Range size: {sys.getsizeof(gen)} bytes")


demo_generators()


# === Lazy Loading Pattern ===
class LazyLoader:
    """Lazy loading - โหลดข้อมูลเฉพาะเมื่อต้องการ"""
    
    def __init__(self):
        self._data = None
    
    @property
    def data(self):
        """โหลด data เมื่อถูกเรียกครั้งแรก"""
        if self._data is None:
            print("Loading data...")
            self._data = [i ** 2 for i in range(1000)]
        return self._data


class LazyDatabase:
    """Database connection ที่ connect เมื่อต้องการ"""
    
    def __init__(self, url: str):
        self._url = url
        self._connection = None
    
    @property
    def connection(self):
        if self._connection is None:
            print(f"Connecting to {self._url}...")
            # self._connection = create_engine(self._url)
            self._connection = {"status": "connected", "url": self._url}
        return self._connection
    
    def query(self, sql: str):
        return self.connection  # ใช้ lazy connection


loader = LazyLoader()
print("Created LazyLoader (no data loaded yet)")
print(f"Accessing data: first item = {loader.data[0]}")  # โหลดตอนนี้
print(f"Access again: first item = {loader.data[0]}")   # จาก cache
```

---

## 7. Algorithm Optimization

```python
import bisect
from collections import Counter, defaultdict
from itertools import groupby

# === ตัวอย่าง Optimizations ===

# 1. Set lookup O(1) แทน List O(n)
def demo_set_vs_list():
    data = list(range(100000))
    target = 99999
    
    import timeit
    
    list_time = timeit.timeit(lambda: target in data, number=10000)
    
    data_set = set(data)
    set_time = timeit.timeit(lambda: target in data_set, number=10000)
    
    print(f"List lookup: {list_time:.4f}s")
    print(f"Set lookup: {set_time:.6f}s")
    print(f"Set is {list_time/set_time:.0f}x faster")


# 2. Counter สำหรับ counting
def demo_counter():
    words = "the quick brown fox jumps over the lazy dog the fox".split()
    
    # Slow
    counts_dict = {}
    for word in words:
        counts_dict[word] = counts_dict.get(word, 0) + 1
    
    # Fast
    counts = Counter(words)
    
    print(f"Most common: {counts.most_common(3)}")


# 3. bisect สำหรับ binary search ใน sorted list
def demo_bisect():
    sorted_list = list(range(0, 1000000, 2))  # sorted even numbers
    target = 500000
    
    # Binary search O(log n) แทน linear O(n)
    index = bisect.bisect_left(sorted_list, target)
    found = index < len(sorted_list) and sorted_list[index] == target
    
    print(f"Found {target}: {found} at index {index}")
    
    # Insert เพื่อคงความ sorted
    bisect.insort(sorted_list, 500001)
    print(f"Inserted 500001 at correct position")


# 4. defaultdict
def demo_defaultdict():
    data = [("a", 1), ("b", 2), ("a", 3), ("b", 4), ("c", 5)]
    
    # Regular dict - verbose
    result = {}
    for key, val in data:
        if key not in result:
            result[key] = []
        result[key].append(val)
    
    # defaultdict - cleaner
    from collections import defaultdict
    result = defaultdict(list)
    for key, val in data:
        result[key].append(val)
    
    print(f"Grouped: {dict(result)}")


# 5. String operations
def demo_string_ops():
    import timeit
    
    parts = [str(i) for i in range(1000)]
    
    # ❌ String concatenation ใน loop - O(n²)
    bad_time = timeit.timeit(
        'result = ""; [result := result + p for p in parts]',
        globals={"parts": parts},
        number=100
    )
    
    # ✅ join - O(n)
    good_time = timeit.timeit(
        '"".join(parts)',
        globals={"parts": parts},
        number=100
    )
    
    print(f"Concat: {bad_time:.4f}s")
    print(f"Join: {good_time:.6f}s")
    print(f"Join is {bad_time/good_time:.0f}x faster")


demo_set_vs_list()
demo_counter()
demo_bisect()
demo_defaultdict()
demo_string_ops()
```

---

## 8. Concurrency Optimization

```python
import asyncio
import concurrent.futures
import time
from typing import Callable, List, Any

# === ThreadPoolExecutor สำหรับ I/O-bound tasks ===
def io_bound_task(n: int) -> int:
    """Simulate I/O bound work"""
    time.sleep(0.1)  # Simulate network/disk
    return n * 2


def demo_thread_pool():
    n = 10
    
    # Sequential
    start = time.time()
    results_seq = [io_bound_task(i) for i in range(n)]
    seq_time = time.time() - start
    
    # Parallel with ThreadPoolExecutor
    start = time.time()
    with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:
        results_parallel = list(executor.map(io_bound_task, range(n)))
    parallel_time = time.time() - start
    
    print(f"Sequential: {seq_time:.2f}s")
    print(f"Parallel (threads): {parallel_time:.2f}s")
    print(f"Speedup: {seq_time/parallel_time:.1f}x")


# === ProcessPoolExecutor สำหรับ CPU-bound tasks ===
def cpu_bound_task(n: int) -> int:
    """Simulate CPU intensive work"""
    return sum(i ** 2 for i in range(n))


def demo_process_pool():
    tasks = [100000] * 8  # 8 tasks
    
    # Sequential
    start = time.time()
    results_seq = [cpu_bound_task(n) for n in tasks]
    seq_time = time.time() - start
    
    # Parallel with ProcessPoolExecutor
    start = time.time()
    with concurrent.futures.ProcessPoolExecutor(max_workers=4) as executor:
        results_parallel = list(executor.map(cpu_bound_task, tasks))
    parallel_time = time.time() - start
    
    print(f"\nCPU-bound Sequential: {seq_time:.2f}s")
    print(f"CPU-bound Parallel (processes): {parallel_time:.2f}s")
    print(f"Speedup: {seq_time/parallel_time:.1f}x")


# === asyncio สำหรับ Async I/O ===
async def async_io_task(n: int) -> int:
    """Async I/O task"""
    await asyncio.sleep(0.1)  # Async sleep
    return n * 2


async def demo_asyncio():
    n = 20
    
    # Sequential async
    start = time.time()
    results = []
    for i in range(n):
        result = await async_io_task(i)
        results.append(result)
    seq_time = time.time() - start
    
    # Concurrent async
    start = time.time()
    results = await asyncio.gather(*[async_io_task(i) for i in range(n)])
    concurrent_time = time.time() - start
    
    print(f"\nAsync Sequential: {seq_time:.2f}s")
    print(f"Async Concurrent: {concurrent_time:.2f}s")
    print(f"Speedup: {seq_time/concurrent_time:.1f}x")


# Run demos
demo_thread_pool()
# demo_process_pool()  # Uncomment เพื่อทดสอบ (ต้องการ if __name__ == '__main__')
asyncio.run(demo_asyncio())
```

---

## 9. สรุป Part 048

✅ **cProfile** - Profile ทั้ง function, ดู bottlenecks  
✅ **line_profiler** - Profile ทีละบรรทัด  
✅ **tracemalloc** - Track memory allocations  
✅ **lru_cache** - Memoization, cached_property  
✅ **TTL Cache** - Cache พร้อม expiry  
✅ **Redis Caching** - Distributed cache  
✅ **Generators** - ประหยัด memory  
✅ **Algorithm Optimization** - Set, Counter, bisect  
✅ **Concurrency** - Thread/Process pools, asyncio  

**Optimization Principles:**
1. Measure first, optimize second
2. Profile before optimizing
3. O(1) operations > O(log n) > O(n) > O(n²)
4. Memory vs Speed tradeoffs
5. I/O bound = Threads/Async, CPU bound = Processes

## ➡️ ถัดไป: Part 049 - Data Validation with Pydantic
*Part 048/100+ | Python Course - Beginner to World-Class*
