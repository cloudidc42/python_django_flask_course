# Part 018 - Decorators (ตัวตกแต่งฟังก์ชัน)

## เป้าหมาย
- เข้าใจหลักการทำงานของ Decorators
- สร้าง Function Decorators และ Class Decorators
- ใช้ Built-in Decorators: `@property`, `@classmethod`, `@staticmethod`
- ใช้ `functools.wraps` เพื่อรักษา metadata
- ใช้ Multiple Decorators และ Decorators with Arguments
- สร้าง practical decorators: timer, logger, cache, retry

---

## 1. พื้นฐาน Decorators

Decorator คือ function ที่รับ function มาเป็น argument และคืน function ที่ถูก "ตกแต่ง"

```python
# ก่อน decorator - โค้ดปกติ
def greet():
    return "Hello!"

# Decorator คือ function ที่ wrap function อื่น
def make_uppercase(func):
    """Decorator ที่แปลง output เป็น uppercase"""
    def wrapper():
        result = func()
        return result.upper()
    return wrapper

# วิธีที่ 1: เรียกด้วยมือ
greet_upper = make_uppercase(greet)
print(greet_upper())  # HELLO!

# วิธีที่ 2: ใช้ @ syntax (Syntactic Sugar)
@make_uppercase
def greet():
    return "Hello World!"

print(greet())  # HELLO WORLD!

# ตัวอย่างง่ายๆ เพิ่มเติม
def add_border(func):
    """เพิ่ม border รอบๆ output"""
    def wrapper():
        print("=" * 30)
        result = func()
        print("=" * 30)
        return result
    return wrapper

@add_border
def show_message():
    print("Important message!")
    return "done"

show_message()
```

---

## 2. Decorators กับ Arguments

```python
import functools

# Decorator ที่รองรับ arguments ใดก็ได้
def logger(func):
    """Decorator สำหรับ log การเรียกฟังก์ชัน"""
    
    @functools.wraps(func)  # รักษา metadata ของ func เดิม
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__} with args={args}, kwargs={kwargs}")
        result = func(*args, **kwargs)
        print(f"{func.__name__} returned: {result}")
        return result
    
    return wrapper


@logger
def add(a: int, b: int) -> int:
    """บวกสองเลข"""
    return a + b


@logger
def greet(name: str, greeting: str = "Hello") -> str:
    """ทักทาย"""
    return f"{greeting}, {name}!"


add(3, 5)
# Calling add with args=(3, 5), kwargs={}
# add returned: 8

greet("Alice", greeting="Hi")
# Calling greet with args=('Alice',), kwargs={'greeting': 'Hi'}
# greet returned: Hi, Alice!

# ตรวจสอบว่า metadata ยังคงอยู่ (เพราะใช้ functools.wraps)
print(add.__name__)   # add (ไม่ใช่ wrapper)
print(add.__doc__)    # บวกสองเลข
```

---

## 3. functools.wraps

```python
import functools

# ปัญหาโดยไม่ใช้ functools.wraps
def bad_decorator(func):
    def wrapper(*args, **kwargs):
        """This is wrapper's docstring"""
        return func(*args, **kwargs)
    return wrapper

@bad_decorator
def my_function():
    """This is my_function's docstring"""
    pass

print(my_function.__name__)  # wrapper (ผิด! ควรเป็น my_function)
print(my_function.__doc__)   # This is wrapper's docstring (ผิด!)

# แก้ด้วย functools.wraps
def good_decorator(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        """This is wrapper's docstring"""
        return func(*args, **kwargs)
    return wrapper

@good_decorator
def my_function():
    """This is my_function's docstring"""
    pass

print(my_function.__name__)  # my_function (ถูกต้อง!)
print(my_function.__doc__)   # This is my_function's docstring (ถูกต้อง!)

# functools.wraps คัดลอก attributes เหล่านี้:
# __name__, __qualname__, __doc__, __dict__, __module__, __annotations__, __wrapped__
```

---

## 4. Decorators with Arguments (Parametrized Decorators)

```python
import functools
import time

def retry(max_attempts: int = 3, delay: float = 1.0, 
          exceptions: tuple = (Exception,)):
    """
    Decorator ที่ retry เมื่อเกิด error
    
    Args:
        max_attempts: จำนวนครั้งสูงสุดที่จะลองใหม่
        delay: เวลารอระหว่าง retry (วินาที)
        exceptions: ประเภท exception ที่จะ retry
    """
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    last_exception = e
                    print(f"Attempt {attempt}/{max_attempts} failed: {e}")
                    if attempt < max_attempts:
                        time.sleep(delay)
            
            raise last_exception
        
        return wrapper
    return decorator


# ใช้ retry decorator
@retry(max_attempts=3, delay=0.1, exceptions=(ValueError, ConnectionError))
def unstable_function(n: int) -> str:
    """ฟังก์ชันที่อาจ fail"""
    import random
    if random.random() < 0.7:  # fail 70% ของเวลา
        raise ValueError(f"Random failure for {n}")
    return f"Success with {n}"

try:
    result = unstable_function(42)
    print(f"Result: {result}")
except ValueError as e:
    print(f"All attempts failed: {e}")


def validate(**validators):
    """
    Decorator สำหรับ validate arguments
    
    Usage:
        @validate(name=str, age=int)
        def create_user(name, age):
            ...
    """
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            # ดึง parameter names ของ function
            import inspect
            sig = inspect.signature(func)
            params = list(sig.parameters.keys())
            
            # สร้าง dict ของ args
            bound_args = {}
            for i, arg in enumerate(args):
                if i < len(params):
                    bound_args[params[i]] = arg
            bound_args.update(kwargs)
            
            # Validate
            for param_name, expected_type in validators.items():
                if param_name in bound_args:
                    value = bound_args[param_name]
                    if not isinstance(value, expected_type):
                        raise TypeError(
                            f"Parameter '{param_name}' expected {expected_type.__name__}, "
                            f"got {type(value).__name__}"
                        )
            
            return func(*args, **kwargs)
        return wrapper
    return decorator


@validate(name=str, age=int, email=str)
def create_user(name: str, age: int, email: str) -> dict:
    """สร้าง user"""
    return {"name": name, "age": age, "email": email}


print(create_user("Alice", 25, "alice@example.com"))  # OK
try:
    create_user("Bob", "twenty", "bob@example.com")  # TypeError
except TypeError as e:
    print(f"Validation error: {e}")
```

---

## 5. Built-in Decorators

### @property

```python
class Temperature:
    """คลาสจัดการอุณหภูมิ"""
    
    def __init__(self, celsius: float = 0):
        self._celsius = celsius  # private attribute
    
    @property
    def celsius(self) -> float:
        """อ่านอุณหภูมิในองศาเซลเซียส"""
        return self._celsius
    
    @celsius.setter
    def celsius(self, value: float):
        """ตั้งค่าอุณหภูมิในองศาเซลเซียส พร้อม validate"""
        if value < -273.15:
            raise ValueError(f"Temperature below absolute zero: {value}")
        self._celsius = value
    
    @celsius.deleter
    def celsius(self):
        """รีเซ็ตอุณหภูมิ"""
        del self._celsius
    
    @property
    def fahrenheit(self) -> float:
        """แปลงเป็นองศาฟาเรนไฮต์"""
        return self._celsius * 9/5 + 32
    
    @fahrenheit.setter
    def fahrenheit(self, value: float):
        """ตั้งค่าในองศาฟาเรนไฮต์"""
        self.celsius = (value - 32) * 5/9
    
    @property
    def kelvin(self) -> float:
        """แปลงเป็นเคลวิน"""
        return self._celsius + 273.15
    
    @property
    def is_freezing(self) -> bool:
        """ตรวจสอบว่าน้ำแข็งได้ไหม"""
        return self._celsius <= 0
    
    @property
    def is_boiling(self) -> bool:
        """ตรวจสอบว่าน้ำเดือดไหม"""
        return self._celsius >= 100
    
    def __str__(self):
        return f"{self._celsius:.1f}°C"
    
    def __repr__(self):
        return f"Temperature({self._celsius!r})"


# ทดสอบ @property
temp = Temperature(25)
print(temp.celsius)      # 25
print(temp.fahrenheit)   # 77.0
print(temp.kelvin)       # 298.15
print(temp.is_freezing)  # False

temp.celsius = 100
print(temp.is_boiling)   # True

temp.fahrenheit = 32  # ตั้งค่าผ่าน fahrenheit setter
print(temp.celsius)    # 0.0
print(temp.is_freezing) # True

try:
    temp.celsius = -300  # ValueError!
except ValueError as e:
    print(f"Error: {e}")
```

### @classmethod และ @staticmethod

```python
from datetime import date
from typing import ClassVar, List
import json

class User:
    """คลาส User"""
    
    # Class variable
    _count: ClassVar[int] = 0
    _users: ClassVar[List] = []
    
    def __init__(self, username: str, email: str, age: int):
        self.username = username
        self.email = email
        self.age = age
        self.created_at = date.today()
        
        # อัปเดต class-level counter
        User._count += 1
        User._users.append(self)
    
    @classmethod
    def from_dict(cls, data: dict) -> "User":
        """
        Alternative constructor จาก dictionary
        ใช้ cls แทน class name เพื่อรองรับ inheritance
        """
        return cls(
            username=data["username"],
            email=data["email"],
            age=data["age"]
        )
    
    @classmethod
    def from_json(cls, json_str: str) -> "User":
        """Alternative constructor จาก JSON string"""
        data = json.loads(json_str)
        return cls.from_dict(data)
    
    @classmethod
    def get_count(cls) -> int:
        """จำนวน user ทั้งหมด"""
        return cls._count
    
    @classmethod
    def get_all(cls) -> List["User"]:
        """ดู user ทั้งหมด"""
        return cls._users.copy()
    
    @classmethod
    def find_by_email(cls, email: str) -> "User":
        """ค้นหา user ด้วย email"""
        for user in cls._users:
            if user.email == email:
                return user
        return None
    
    @staticmethod
    def validate_email(email: str) -> bool:
        """
        Validate รูปแบบ email
        @staticmethod ไม่รับ self หรือ cls
        ใช้สำหรับ utility functions ที่เกี่ยวข้องกับ class
        """
        import re
        pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        return bool(re.match(pattern, email))
    
    @staticmethod
    def hash_password(password: str) -> str:
        """Hash password"""
        import hashlib
        return hashlib.sha256(password.encode()).hexdigest()[:16] + "..."
    
    @staticmethod
    def is_valid_age(age: int) -> bool:
        """ตรวจสอบอายุ"""
        return 0 < age < 150
    
    def __repr__(self):
        return f"User(username={self.username!r}, email={self.email!r})"


# ทดสอบ classmethod และ staticmethod
# Regular constructor
u1 = User("alice", "alice@example.com", 25)

# Alternative constructors
u2 = User.from_dict({"username": "bob", "email": "bob@example.com", "age": 30})

json_data = '{"username": "charlie", "email": "charlie@example.com", "age": 28}'
u3 = User.from_json(json_data)

print(f"Total users: {User.get_count()}")  # 3

# Static methods - ไม่ต้องการ instance
print(User.validate_email("test@example.com"))  # True
print(User.validate_email("not-an-email"))       # False
print(User.hash_password("secret123"))

# ค้นหา user
found = User.find_by_email("bob@example.com")
print(found)  # User(username='bob', email='bob@example.com')


# Class Inheritance กับ classmethod
class AdminUser(User):
    def __init__(self, username: str, email: str, age: int, admin_level: int = 1):
        super().__init__(username, email, age)
        self.admin_level = admin_level
    
    def __repr__(self):
        return f"AdminUser(username={self.username!r}, level={self.admin_level})"


# from_dict จะสร้าง AdminUser ไม่ใช่ User (เพราะใช้ cls)
admin = AdminUser.from_dict({
    "username": "admin",
    "email": "admin@example.com",
    "age": 35
})
print(type(admin))  # <class '__main__.AdminUser'>
print(admin)        # AdminUser(username='admin', level=1)
```

---

## 6. Class Decorators

```python
import functools
from typing import Any, Callable, Type

# Class ที่ใช้เป็น Decorator
class Timer:
    """Decorator class สำหรับวัดเวลา"""
    
    def __init__(self, func: Callable):
        functools.update_wrapper(self, func)
        self.func = func
        self.call_count = 0
        self.total_time = 0.0
    
    def __call__(self, *args, **kwargs):
        import time
        start = time.perf_counter()
        result = self.func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        
        self.call_count += 1
        self.total_time += elapsed
        
        print(f"{self.func.__name__} took {elapsed:.4f}s "
              f"(call #{self.call_count}, avg: {self.total_time/self.call_count:.4f}s)")
        return result
    
    def reset_stats(self):
        """รีเซ็ตสถิติ"""
        self.call_count = 0
        self.total_time = 0.0


@Timer
def slow_function(n: int) -> int:
    """ฟังก์ชันที่ช้า"""
    import time
    time.sleep(0.01 * n)
    return sum(range(n * 1000))


slow_function(1)
slow_function(2)
slow_function(3)
print(f"Called {slow_function.call_count} times, total: {slow_function.total_time:.4f}s")


# Decorator สำหรับ class (ไม่ใช่ function)
def singleton(cls: Type) -> Type:
    """
    Decorator ที่ทำให้ class เป็น Singleton
    - สร้าง instance ได้แค่ครั้งเดียว
    - ทุกครั้งที่เรียก Class() จะได้ instance เดิม
    """
    instances = {}
    
    @functools.wraps(cls)
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    
    return get_instance


@singleton
class DatabaseConnection:
    """Database connection - มีแค่ connection เดียว"""
    
    def __init__(self, host: str = "localhost"):
        self.host = host
        self.is_connected = False
        print(f"Creating new DatabaseConnection to {host}")
    
    def connect(self):
        self.is_connected = True
        print(f"Connected to {self.host}")


# ทดสอบ singleton
db1 = DatabaseConnection("server1")  # สร้างใหม่
db2 = DatabaseConnection("server2")  # ได้ instance เดิม!
print(db1 is db2)  # True
print(db1.host)    # server1 (ไม่ใช่ server2)


def add_repr(cls: Type) -> Type:
    """Decorator เพิ่ม __repr__ อัตโนมัติ"""
    
    def __repr__(self) -> str:
        attrs = ", ".join(
            f"{k}={v!r}" 
            for k, v in vars(self).items() 
            if not k.startswith("_")
        )
        return f"{cls.__name__}({attrs})"
    
    cls.__repr__ = __repr__
    return cls


@add_repr
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y


p = Point(3, 4)
print(p)  # Point(x=3, y=4)
```

---

## 7. Practical Decorators

### Timer Decorator

```python
import time
import functools
from typing import Callable, Optional

def timer(unit: str = "ms", verbose: bool = True):
    """
    Decorator วัดเวลาการทำงาน
    
    Args:
        unit: หน่วยเวลา 's', 'ms', 'us'
        verbose: แสดงผลหรือไม่
    """
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            start = time.perf_counter()
            result = func(*args, **kwargs)
            elapsed = time.perf_counter() - start
            
            # แปลงหน่วย
            if unit == "ms":
                display_time = elapsed * 1000
                unit_str = "ms"
            elif unit == "us":
                display_time = elapsed * 1_000_000
                unit_str = "μs"
            else:
                display_time = elapsed
                unit_str = "s"
            
            if verbose:
                print(f"⏱ {func.__name__}: {display_time:.3f}{unit_str}")
            
            wrapper.last_time = elapsed
            return result
        
        wrapper.last_time = 0.0
        return wrapper
    return decorator


@timer(unit="ms")
def sort_large_list(n: int) -> list:
    """เรียงลำดับ list ขนาดใหญ่"""
    import random
    data = [random.randint(0, 1000) for _ in range(n)]
    return sorted(data)


sort_large_list(10000)   # แสดงเวลาเป็น ms
sort_large_list(100000)
```

### Logger Decorator

```python
import logging
import functools
from typing import Callable
from datetime import datetime

# ตั้งค่า logging
logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)


def log_calls(logger_name: Optional[str] = None, 
              level: int = logging.DEBUG,
              log_args: bool = True,
              log_result: bool = True):
    """
    Decorator สำหรับ log การเรียกฟังก์ชัน
    
    Args:
        logger_name: ชื่อ logger (ถ้าไม่ระบุใช้ชื่อ module)
        level: ระดับ log
        log_args: log arguments หรือไม่
        log_result: log ผลลัพธ์หรือไม่
    """
    def decorator(func: Callable) -> Callable:
        logger = logging.getLogger(logger_name or func.__module__)
        
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            if log_args:
                logger.log(level, 
                    f"CALL {func.__name__}(args={args}, kwargs={kwargs})")
            else:
                logger.log(level, f"CALL {func.__name__}()")
            
            try:
                result = func(*args, **kwargs)
                if log_result:
                    logger.log(level, f"RETURN {func.__name__} -> {result!r}")
                return result
            except Exception as e:
                logger.error(f"ERROR in {func.__name__}: {type(e).__name__}: {e}")
                raise
        
        return wrapper
    return decorator


@log_calls(log_args=True, log_result=True)
def calculate(a: float, b: float, operation: str) -> float:
    """คำนวณตาม operation"""
    ops = {
        "add": lambda x, y: x + y,
        "sub": lambda x, y: x - y,
        "mul": lambda x, y: x * y,
        "div": lambda x, y: x / y
    }
    if operation not in ops:
        raise ValueError(f"Unknown operation: {operation}")
    return ops[operation](a, b)


calculate(10, 5, "add")
calculate(10, 0, "div")  # จะ raise exception
```

### Cache Decorator

```python
import functools
import time
from typing import Callable, Optional, Any
from collections import OrderedDict

def memoize(max_size: int = 128, ttl: Optional[float] = None):
    """
    Cache decorator ด้วย LRU eviction และ TTL
    
    Args:
        max_size: จำนวน cache entries สูงสุด
        ttl: time-to-live เป็นวินาที (None = ไม่หมดอายุ)
    """
    def decorator(func: Callable) -> Callable:
        cache: OrderedDict = OrderedDict()
        timestamps: dict = {}
        
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            # สร้าง cache key
            key = (args, tuple(sorted(kwargs.items())))
            
            # ตรวจสอบ TTL
            if key in cache:
                if ttl is not None:
                    age = time.time() - timestamps[key]
                    if age > ttl:
                        del cache[key]
                        del timestamps[key]
                    else:
                        # Cache hit - ย้ายไปท้าย (most recently used)
                        cache.move_to_end(key)
                        return cache[key]
                else:
                    cache.move_to_end(key)
                    return cache[key]
            
            # Cache miss - คำนวณและเก็บ
            result = func(*args, **kwargs)
            cache[key] = result
            timestamps[key] = time.time()
            
            # LRU eviction
            while len(cache) > max_size:
                oldest_key = next(iter(cache))
                del cache[oldest_key]
                if oldest_key in timestamps:
                    del timestamps[oldest_key]
            
            return result
        
        def cache_info():
            return {
                "size": len(cache),
                "max_size": max_size,
                "ttl": ttl
            }
        
        def cache_clear():
            cache.clear()
            timestamps.clear()
        
        wrapper.cache_info = cache_info
        wrapper.cache_clear = cache_clear
        return wrapper
    
    return decorator


# ทดสอบ memoize
@memoize(max_size=100, ttl=60)
def fibonacci(n: int) -> int:
    """Fibonacci ที่ cache ผลลัพธ์"""
    if n <= 1:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)


# ไม่มี cache: O(2^n)
# มี cache: O(n)
import time
start = time.perf_counter()
result = fibonacci(35)
elapsed = time.perf_counter() - start

print(f"fibonacci(35) = {result}")
print(f"Time: {elapsed*1000:.2f}ms")
print(f"Cache info: {fibonacci.cache_info()}")

# Python built-in lru_cache
from functools import lru_cache

@lru_cache(maxsize=None)
def fib(n: int) -> int:
    if n <= 1:
        return n
    return fib(n-1) + fib(n-2)

print(fib(100))
print(fib.cache_info())  # CacheInfo(hits=..., misses=..., maxsize=None, currsize=...)
```

---

## 8. Multiple Decorators

```python
import functools
import time
import logging

# Decorators ถูกใช้จากล่างขึ้นบน
def decorator_a(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        print("A: before")
        result = func(*args, **kwargs)
        print("A: after")
        return result
    return wrapper

def decorator_b(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        print("B: before")
        result = func(*args, **kwargs)
        print("B: after")
        return result
    return wrapper

def decorator_c(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        print("C: before")
        result = func(*args, **kwargs)
        print("C: after")
        return result
    return wrapper


@decorator_a
@decorator_b
@decorator_c
def my_function():
    print("Function executing")

my_function()
# C: before   <- C ถูกใช้ก่อน (ใกล้ function มากสุด)
# B: before
# A: before
# Function executing
# A: after    <- A ถูก "ออก" ก่อน (ห่อรอบนอกสุด)
# B: after
# C: after

# เทียบเท่ากับ
# my_function = decorator_a(decorator_b(decorator_c(my_function)))


# ตัวอย่าง real world: Web route ที่ใช้หลาย decorators
def authenticate(func):
    """ตรวจสอบ authentication"""
    @functools.wraps(func)
    def wrapper(request, *args, **kwargs):
        if not request.get("user"):
            return {"error": "Unauthorized", "code": 401}
        return func(request, *args, **kwargs)
    return wrapper


def authorize(role: str):
    """ตรวจสอบสิทธิ์"""
    def decorator(func):
        @functools.wraps(func)
        def wrapper(request, *args, **kwargs):
            user_role = request.get("user", {}).get("role")
            if user_role != role and user_role != "admin":
                return {"error": "Forbidden", "code": 403}
            return func(request, *args, **kwargs)
        return wrapper
    return decorator


def rate_limit(max_calls: int = 100, period: int = 60):
    """จำกัดการเรียก API"""
    calls = {}
    
    def decorator(func):
        @functools.wraps(func)
        def wrapper(request, *args, **kwargs):
            user_id = request.get("user", {}).get("id", "anonymous")
            now = time.time()
            
            # ล้าง calls ที่เก่าเกิน period
            calls[user_id] = [t for t in calls.get(user_id, []) 
                              if now - t < period]
            
            if len(calls.get(user_id, [])) >= max_calls:
                return {"error": "Rate limit exceeded", "code": 429}
            
            calls.setdefault(user_id, []).append(now)
            return func(request, *args, **kwargs)
        return wrapper
    return decorator


@authenticate
@authorize(role="admin")
@rate_limit(max_calls=10, period=60)
def delete_user(request: dict, user_id: int):
    """ลบ user - ต้องเป็น admin เท่านั้น"""
    return {"success": True, "deleted_user_id": user_id}


# ทดสอบ
req_no_auth = {}
req_user = {"user": {"id": 1, "role": "user"}}
req_admin = {"user": {"id": 0, "role": "admin"}}

print(delete_user(req_no_auth, 5))    # Unauthorized
print(delete_user(req_user, 5))       # Forbidden
print(delete_user(req_admin, 5))      # Success
```

---

## 9. ตัวอย่างจริง: REST API Decorators

```python
import functools
import time
import json
from typing import Callable, Any

# สร้าง mini framework สำหรับ API

class APIError(Exception):
    def __init__(self, message: str, status_code: int = 400):
        self.message = message
        self.status_code = status_code
        super().__init__(message)


def json_response(func: Callable) -> Callable:
    """แปลง return value เป็น JSON"""
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        try:
            result = func(*args, **kwargs)
            return {
                "status": "success",
                "data": result,
                "timestamp": time.time()
            }
        except APIError as e:
            return {
                "status": "error",
                "error": e.message,
                "code": e.status_code,
                "timestamp": time.time()
            }
        except Exception as e:
            return {
                "status": "error",
                "error": "Internal server error",
                "code": 500,
                "timestamp": time.time()
            }
    return wrapper


def require_params(*required_params):
    """ตรวจสอบ required parameters"""
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        def wrapper(params: dict = None, *args, **kwargs):
            params = params or {}
            missing = [p for p in required_params if p not in params]
            if missing:
                raise APIError(
                    f"Missing required parameters: {', '.join(missing)}",
                    status_code=422
                )
            return func(params, *args, **kwargs)
        return wrapper
    return decorator


def cache_response(ttl: int = 300):
    """Cache API response"""
    _cache = {}
    _timestamps = {}
    
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            key = str(args) + str(kwargs)
            
            if key in _cache:
                age = time.time() - _timestamps[key]
                if age < ttl:
                    return {**_cache[key], "from_cache": True}
            
            result = func(*args, **kwargs)
            _cache[key] = result
            _timestamps[key] = time.time()
            return result
        
        return wrapper
    return decorator


# ใช้ decorators กับ API functions
@json_response
@require_params("user_id", "product_id")
@cache_response(ttl=60)
def get_product_for_user(params: dict) -> dict:
    """ดึงข้อมูลสินค้าสำหรับ user"""
    user_id = params["user_id"]
    product_id = params["product_id"]
    
    # Simulate database lookup
    return {
        "product_id": product_id,
        "name": f"Product {product_id}",
        "price": 99.99,
        "user_discount": 0.1 if user_id % 2 == 0 else 0
    }


# ทดสอบ
result = get_product_for_user({"user_id": 1, "product_id": 42})
print(json.dumps(result, indent=2))

# missing params
result = get_product_for_user({"user_id": 1})
print(json.dumps(result, indent=2))
```

---

## Exercises

### Exercise 1: Rate Limiter
สร้าง `@rate_limit(calls_per_second=5)` decorator ที่:
- จำกัดการเรียกฟังก์ชัน
- Auto-sleep เมื่อเกิน limit
- นับจำนวน calls ที่ถูก throttle

### Exercise 2: Input/Output Transformer
สร้าง decorators:
- `@normalize_input` แปลง string input เป็น lowercase, strip whitespace
- `@format_output(template)` format output ตาม template
- `@measure_memory` วัด memory ที่ใช้

### Exercise 3: Property Descriptor
สร้าง `@validated_property(min_val, max_val)` decorator ที่ทำงานเหมือน `@property` แต่ validate ค่าอัตโนมัติ:
```python
class Circle:
    @validated_property(min_val=0)
    def radius(self):
        return self._radius
```

---

## สรุป

| Decorator | คำอธิบาย | ตัวอย่าง |
|-----------|---------|---------|
| Function decorator | ห่อฟังก์ชัน | `def dec(func): ...` |
| Class decorator | ห่อคลาส | `def dec(cls): ...` |
| Decorator class | ใช้ class เป็น decorator | `class Dec: def __call__` |
| `@property` | getter/setter/deleter | `@prop.setter` |
| `@classmethod` | เมธอดระดับ class | `cls` แทน `self` |
| `@staticmethod` | utility method | ไม่รับ self/cls |
| `@functools.wraps` | รักษา metadata | ใส่ใน wrapper ทุกครั้ง |
| `@lru_cache` | built-in cache | `functools.lru_cache` |

---

## ต่อไป

[Part 019 - Generators and Iterators](part-019.md) - yield, custom iterators, generator expressions
