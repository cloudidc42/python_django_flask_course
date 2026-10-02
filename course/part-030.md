# Part 030: Design Patterns เบื้องต้น
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Design Patterns และทำไมถึงสำคัญ
- Creational Patterns: Singleton, Factory
- Behavioral Patterns: Observer, Strategy
- Structural Patterns: Decorator pattern (ไม่ใช่ syntax)
- ตัวอย่างจริงที่ใช้ใน Python projects

---

## 1. Design Patterns คืออะไร?

```python
# Design Patterns = วิธีแก้ปัญหาที่พิสูจน์แล้วว่าใช้ได้ดี
# ไม่ใช่โค้ดที่ copy วาง แต่เป็น template/แนวคิด

# แบ่งเป็น 3 กลุ่มหลัก:
patterns = {
    "Creational": [
        "Singleton - มี instance เดียวในโปรแกรม",
        "Factory - สร้าง objects โดยไม่ระบุ class ตรงๆ",
        "Builder - สร้าง complex objects ทีละขั้น",
        "Prototype - clone objects",
    ],
    "Structural": [
        "Decorator - เพิ่ม behavior โดยไม่แก้ class เดิม",
        "Adapter - ทำให้ interfaces ที่ไม่ compatible ใช้ด้วยกันได้",
        "Facade - simplify complex subsystem",
        "Composite - tree structures",
    ],
    "Behavioral": [
        "Observer - notify objects เมื่อ state เปลี่ยน",
        "Strategy - เลือก algorithm ที่ runtime",
        "Command - encapsulate operations เป็น objects",
        "Iterator - traverse collections",
    ]
}

for category, items in patterns.items():
    print(f"\n{category} Patterns:")
    for item in items:
        print(f"  • {item}")
```

---

## 2. Singleton Pattern

```python
# Singleton: ทำให้ class มีได้แค่ 1 instance ตลอดโปรแกรม
# ใช้เมื่อ: config, database connection, logger, cache

# Method 1: ใช้ __new__
class DatabaseConnection:
    _instance = None
    _initialized = False
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
    
    def __init__(self):
        # Guard: ไม่ให้ init ซ้ำ
        if not DatabaseConnection._initialized:
            self.host = "localhost"
            self.port = 5432
            self.connected = False
            print("Database connection initialized")
            DatabaseConnection._initialized = True
    
    def connect(self) -> None:
        self.connected = True
        print(f"Connected to {self.host}:{self.port}")
    
    def query(self, sql: str) -> list:
        if not self.connected:
            raise RuntimeError("Not connected!")
        print(f"Executing: {sql}")
        return []

# ทดสอบ Singleton
db1 = DatabaseConnection()
db2 = DatabaseConnection()
db3 = DatabaseConnection()

print(db1 is db2)    # True - same object!
print(db1 is db3)    # True - same object!
print(id(db1) == id(db2))  # True

db1.connect()
print(db2.connected)  # True - เพราะ db1 และ db2 เป็น object เดียวกัน

# Method 2: ใช้ Decorator
def singleton(cls):
    """Decorator สำหรับทำ class เป็น Singleton"""
    instances = {}
    
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    
    return get_instance

@singleton
class AppConfig:
    def __init__(self):
        self.debug = False
        self.version = "1.0.0"
        self.max_connections = 10
        print("AppConfig initialized")

config1 = AppConfig()
config2 = AppConfig()
print(config1 is config2)  # True
print(config2.version)     # 1.0.0

# Method 3: ใช้ metaclass (advanced)
class SingletonMeta(type):
    _instances = {}
    
    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class Logger(metaclass=SingletonMeta):
    def __init__(self):
        self.logs = []
        print("Logger initialized")
    
    def log(self, message: str) -> None:
        self.logs.append(message)
        print(f"[LOG] {message}")
    
    def get_logs(self) -> list:
        return self.logs.copy()

log1 = Logger()
log2 = Logger()
log1.log("User logged in")
print(len(log2.get_logs()))  # 1 - เพราะ log1 และ log2 เป็น object เดียวกัน

# Thread-safe Singleton
import threading

class ThreadSafeSingleton:
    _instance = None
    _lock = threading.Lock()
    
    def __new__(cls):
        if cls._instance is None:
            with cls._lock:
                # Double-checked locking
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
        return cls._instance
```

---

## 3. Factory Pattern

```python
# Factory: สร้าง objects โดยไม่ระบุ class ตรงๆ ใน client code
# ใช้เมื่อ: ไม่รู้ล่วงหน้าว่าจะสร้าง object ชนิดไหน

from abc import ABC, abstractmethod
from typing import Dict, Type

# Product interface
class Notification(ABC):
    @abstractmethod
    def send(self, recipient: str, message: str) -> bool: ...
    
    @abstractmethod
    def get_type(self) -> str: ...

# Concrete products
class EmailNotification(Notification):
    def send(self, recipient: str, message: str) -> bool:
        print(f"📧 Email to {recipient}: {message}")
        return True
    
    def get_type(self) -> str:
        return "email"

class SMSNotification(Notification):
    def send(self, recipient: str, message: str) -> bool:
        print(f"📱 SMS to {recipient}: {message}")
        return True
    
    def get_type(self) -> str:
        return "sms"

class PushNotification(Notification):
    def send(self, recipient: str, message: str) -> bool:
        print(f"🔔 Push to {recipient}: {message}")
        return True
    
    def get_type(self) -> str:
        return "push"

class LineNotification(Notification):
    def send(self, recipient: str, message: str) -> bool:
        print(f"💬 LINE to {recipient}: {message}")
        return True
    
    def get_type(self) -> str:
        return "line"

# Simple Factory
class NotificationFactory:
    _registry: Dict[str, Type[Notification]] = {
        "email": EmailNotification,
        "sms": SMSNotification,
        "push": PushNotification,
        "line": LineNotification,
    }
    
    @classmethod
    def create(cls, notification_type: str) -> Notification:
        NotifClass = cls._registry.get(notification_type.lower())
        if not NotifClass:
            raise ValueError(
                f"Unknown notification type: {notification_type}\n"
                f"Available: {', '.join(cls._registry.keys())}"
            )
        return NotifClass()
    
    @classmethod
    def register(cls, name: str, notification_class: Type[Notification]) -> None:
        """ลงทะเบียน notification type ใหม่"""
        cls._registry[name] = notification_class

# ใช้งาน Factory
def send_notification(notif_type: str, recipient: str, message: str) -> None:
    notif = NotificationFactory.create(notif_type)
    notif.send(recipient, message)

send_notification("email", "alice@example.com", "Welcome!")
send_notification("sms", "+66812345678", "OTP: 123456")
send_notification("push", "device_token_xyz", "New message!")

# เพิ่ม notification type ใหม่
class TelegramNotification(Notification):
    def send(self, recipient: str, message: str) -> bool:
        print(f"📨 Telegram to @{recipient}: {message}")
        return True
    
    def get_type(self) -> str:
        return "telegram"

NotificationFactory.register("telegram", TelegramNotification)
send_notification("telegram", "alice_tg", "Hello from Telegram!")

# Abstract Factory: กลุ่ม objects ที่เกี่ยวข้องกัน
class UITheme(ABC):
    @abstractmethod
    def create_button(self) -> str: ...
    
    @abstractmethod
    def create_input(self) -> str: ...
    
    @abstractmethod
    def create_card(self) -> str: ...

class LightTheme(UITheme):
    def create_button(self) -> str:
        return "⬜ [Light Button]"
    
    def create_input(self) -> str:
        return "⬜ [Light Input Field]"
    
    def create_card(self) -> str:
        return "⬜ [Light Card]"

class DarkTheme(UITheme):
    def create_button(self) -> str:
        return "⬛ [Dark Button]"
    
    def create_input(self) -> str:
        return "⬛ [Dark Input Field]"
    
    def create_card(self) -> str:
        return "⬛ [Dark Card]"

def render_ui(theme: UITheme) -> None:
    print(theme.create_card())
    print(theme.create_input())
    print(theme.create_button())

print("\n=== Light Theme ===")
render_ui(LightTheme())
print("\n=== Dark Theme ===")
render_ui(DarkTheme())
```

---

## 4. Observer Pattern

```python
# Observer: เมื่อ object (subject) เปลี่ยน state, 
# notify objects อื่นๆ (observers) อัตโนมัติ
# ใช้เมื่อ: event systems, UI updates, pub/sub

from abc import ABC, abstractmethod
from typing import List, Dict, Any
from dataclasses import dataclass, field

# Observer interface
class Observer(ABC):
    @abstractmethod
    def update(self, event: str, data: Any) -> None: ...

# Subject (Observable)
class Subject:
    def __init__(self) -> None:
        self._observers: Dict[str, List[Observer]] = {}
    
    def subscribe(self, event: str, observer: Observer) -> None:
        if event not in self._observers:
            self._observers[event] = []
        self._observers[event].append(observer)
        print(f"  [{observer.__class__.__name__}] subscribed to '{event}'")
    
    def unsubscribe(self, event: str, observer: Observer) -> None:
        if event in self._observers:
            self._observers[event].remove(observer)
    
    def notify(self, event: str, data: Any = None) -> None:
        """Notify ทุก observers ที่ subscribe event นี้"""
        observers = self._observers.get(event, [])
        for obs in observers:
            obs.update(event, data)

# Concrete Subject
@dataclass
class ShoppingCart(Subject):
    user_id: str
    items: List[Dict] = field(default_factory=list)
    
    def add_item(self, product: str, price: float, qty: int = 1) -> None:
        item = {"product": product, "price": price, "qty": qty}
        self.items.append(item)
        self.notify("item_added", item)
    
    def remove_item(self, product: str) -> None:
        self.items = [i for i in self.items if i["product"] != product]
        self.notify("item_removed", {"product": product})
    
    @property
    def total(self) -> float:
        return sum(i["price"] * i["qty"] for i in self.items)
    
    def checkout(self) -> None:
        self.notify("checkout", {"user": self.user_id, "total": self.total, "items": self.items})
        self.items = []

# Concrete Observers
class EmailService(Observer):
    def update(self, event: str, data: Any) -> None:
        if event == "checkout":
            print(f"📧 Email sent: Order confirmed, total={data['total']:.2f} THB")

class InventoryService(Observer):
    def update(self, event: str, data: Any) -> None:
        if event == "item_added":
            print(f"📦 Inventory: Reserved {data['qty']}x {data['product']}")
        elif event == "item_removed":
            print(f"📦 Inventory: Released {data['product']}")
        elif event == "checkout":
            for item in data["items"]:
                print(f"📦 Inventory: Deducted {item['qty']}x {item['product']}")

class AnalyticsService(Observer):
    def __init__(self):
        self.events = []
    
    def update(self, event: str, data: Any) -> None:
        self.events.append({"event": event, "data": data})
        print(f"📊 Analytics: tracked '{event}'")

class RecommendationService(Observer):
    def update(self, event: str, data: Any) -> None:
        if event == "item_added":
            print(f"💡 Recommendation: Users who bought {data['product']} also bought...")

# ใช้งาน
print("=== Shopping Cart Demo ===")
cart = ShoppingCart(user_id="USER001")

email_svc = EmailService()
inventory_svc = InventoryService()
analytics = AnalyticsService()
reco_svc = RecommendationService()

# Subscribe to events
cart.subscribe("item_added", inventory_svc)
cart.subscribe("item_added", analytics)
cart.subscribe("item_added", reco_svc)
cart.subscribe("item_removed", inventory_svc)
cart.subscribe("checkout", email_svc)
cart.subscribe("checkout", inventory_svc)
cart.subscribe("checkout", analytics)

print("\n--- Adding items ---")
cart.add_item("Python Book", 590.0)
cart.add_item("Mechanical Keyboard", 2500.0, qty=1)

print(f"\nCart total: {cart.total:.2f} THB")

print("\n--- Checkout ---")
cart.checkout()

print(f"\nAnalytics tracked {len(analytics.events)} events")
```

---

## 5. Decorator Pattern (ไม่ใช่ Decorator syntax)

```python
# Decorator Pattern: เพิ่ม behavior ให้ object โดยไม่แก้ class เดิม
# ต่างจาก Python @decorator syntax
# ใช้เมื่อ: add logging, caching, validation, rate limiting

from abc import ABC, abstractmethod
from typing import Optional
import time
import functools

# Component interface
class DataReader(ABC):
    @abstractmethod
    def read(self, key: str) -> Optional[str]: ...

# Concrete Component
class FileDataReader(DataReader):
    def __init__(self, filepath: str) -> None:
        self.filepath = filepath
        # Simulate file data
        self._data = {
            "config": "debug=false\nport=8080",
            "users": "alice,bob,charlie",
            "settings": "theme=dark\nlang=th",
        }
    
    def read(self, key: str) -> Optional[str]:
        print(f"  [FileReader] Reading '{key}' from {self.filepath}")
        time.sleep(0.1)  # Simulate file I/O
        return self._data.get(key)

# Base Decorator
class DataReaderDecorator(DataReader):
    def __init__(self, reader: DataReader) -> None:
        self._reader = reader  # wrapped component
    
    def read(self, key: str) -> Optional[str]:
        return self._reader.read(key)

# Concrete Decorators
class CachingDecorator(DataReaderDecorator):
    """เพิ่ม caching"""
    
    def __init__(self, reader: DataReader) -> None:
        super().__init__(reader)
        self._cache: dict = {}
        self._hits = 0
        self._misses = 0
    
    def read(self, key: str) -> Optional[str]:
        if key in self._cache:
            self._hits += 1
            print(f"  [Cache] HIT '{key}'")
            return self._cache[key]
        
        self._misses += 1
        print(f"  [Cache] MISS '{key}' - fetching...")
        value = self._reader.read(key)
        if value is not None:
            self._cache[key] = value
        return value
    
    def stats(self) -> dict:
        total = self._hits + self._misses
        rate = (self._hits / total * 100) if total > 0 else 0
        return {"hits": self._hits, "misses": self._misses, "hit_rate": f"{rate:.1f}%"}

class LoggingDecorator(DataReaderDecorator):
    """เพิ่ม logging"""
    
    def read(self, key: str) -> Optional[str]:
        start = time.time()
        print(f"  [Logger] Reading key='{key}'")
        value = self._reader.read(key)
        elapsed = (time.time() - start) * 1000
        status = "found" if value else "not found"
        print(f"  [Logger] '{key}' {status} in {elapsed:.1f}ms")
        return value

class ValidationDecorator(DataReaderDecorator):
    """เพิ่ม validation"""
    
    ALLOWED_KEYS = {"config", "users", "settings"}
    
    def read(self, key: str) -> Optional[str]:
        if not key or not isinstance(key, str):
            raise ValueError("Key must be a non-empty string")
        if key not in self.ALLOWED_KEYS:
            raise PermissionError(f"Access denied for key: '{key}'")
        return self._reader.read(key)

# สร้าง decorator chain
print("=== Decorator Pattern Demo ===")
base_reader = FileDataReader("data.txt")

# Stack decorators: Validation → Logging → Cache → File
reader = CachingDecorator(
    LoggingDecorator(
        ValidationDecorator(
            base_reader
        )
    )
)

print("\n--- First reads ---")
v1 = reader.read("config")
print(f"Result: {v1[:20]}...")

v2 = reader.read("users")
print(f"Result: {v2}")

print("\n--- Second reads (should hit cache) ---")
v3 = reader.read("config")  # Cache hit!
v4 = reader.read("users")   # Cache hit!

print("\n--- Access denied ---")
try:
    reader.read("secrets")  # ValidationDecorator จะ reject
except PermissionError as e:
    print(f"  Error: {e}")

# Cache stats
print("\n--- Cache Stats ---")
stats = reader.stats()  # reader is CachingDecorator
print(f"Hits: {stats['hits']}, Misses: {stats['misses']}, Rate: {stats['hit_rate']}")
```

---

## 6. Strategy Pattern

```python
# Strategy: กำหนด algorithms ใน separate classes
# และสามารถ swap algorithms ได้ที่ runtime
# ใช้เมื่อ: sorting, payment, compression, authentication

from abc import ABC, abstractmethod
from typing import List, Callable
import math

# Strategy interface
class SortStrategy(ABC):
    @abstractmethod
    def sort(self, data: List[int]) -> List[int]: ...
    
    @property
    @abstractmethod
    def name(self) -> str: ...

# Concrete Strategies
class BubbleSort(SortStrategy):
    @property
    def name(self) -> str:
        return "Bubble Sort"
    
    def sort(self, data: List[int]) -> List[int]:
        arr = data.copy()
        n = len(arr)
        for i in range(n):
            for j in range(n - i - 1):
                if arr[j] > arr[j + 1]:
                    arr[j], arr[j + 1] = arr[j + 1], arr[j]
        return arr

class QuickSort(SortStrategy):
    @property
    def name(self) -> str:
        return "Quick Sort"
    
    def sort(self, data: List[int]) -> List[int]:
        if len(data) <= 1:
            return data.copy()
        pivot = data[len(data) // 2]
        left = [x for x in data if x < pivot]
        middle = [x for x in data if x == pivot]
        right = [x for x in data if x > pivot]
        return self.sort(left) + middle + self.sort(right)

class MergeSort(SortStrategy):
    @property
    def name(self) -> str:
        return "Merge Sort"
    
    def sort(self, data: List[int]) -> List[int]:
        if len(data) <= 1:
            return data.copy()
        mid = len(data) // 2
        left = self.sort(data[:mid])
        right = self.sort(data[mid:])
        return self._merge(left, right)
    
    def _merge(self, left: List[int], right: List[int]) -> List[int]:
        result = []
        i = j = 0
        while i < len(left) and j < len(right):
            if left[i] <= right[j]:
                result.append(left[i])
                i += 1
            else:
                result.append(right[j])
                j += 1
        return result + left[i:] + right[j:]

# Context
class DataSorter:
    def __init__(self, strategy: SortStrategy) -> None:
        self._strategy = strategy
    
    def set_strategy(self, strategy: SortStrategy) -> None:
        print(f"Switching to {strategy.name}")
        self._strategy = strategy
    
    def sort(self, data: List[int]) -> List[int]:
        start = time.time()
        result = self._strategy.sort(data)
        elapsed = (time.time() - start) * 1000
        print(f"[{self._strategy.name}] sorted {len(data)} items in {elapsed:.2f}ms")
        return result

# ใช้งาน
import random, time

data = random.sample(range(1000), 20)
print(f"Original: {data}")

sorter = DataSorter(BubbleSort())
sorted1 = sorter.sort(data)

sorter.set_strategy(QuickSort())
sorted2 = sorter.sort(data)

sorter.set_strategy(MergeSort())
sorted3 = sorter.sort(data)

print(f"Sorted:   {sorted1}")
print(f"All same: {sorted1 == sorted2 == sorted3}")

# Strategy กับ Discount Calculation (ตัวอย่างจริง)
class DiscountStrategy(ABC):
    @abstractmethod
    def calculate(self, original_price: float) -> float: ...
    
    @abstractmethod
    def description(self) -> str: ...

class NoDiscount(DiscountStrategy):
    def calculate(self, price: float) -> float:
        return price
    
    def description(self) -> str:
        return "No discount"

class PercentageDiscount(DiscountStrategy):
    def __init__(self, percent: float) -> None:
        self.percent = percent
    
    def calculate(self, price: float) -> float:
        return price * (1 - self.percent / 100)
    
    def description(self) -> str:
        return f"{self.percent}% off"

class FlatDiscount(DiscountStrategy):
    def __init__(self, amount: float) -> None:
        self.amount = amount
    
    def calculate(self, price: float) -> float:
        return max(0, price - self.amount)
    
    def description(self) -> str:
        return f"{self.amount:.0f} THB off"

class BuyTwoGetOneFree(DiscountStrategy):
    def calculate(self, price: float) -> float:
        # สมมติ price = ราคารวม 3 ชิ้น
        return (price / 3) * 2
    
    def description(self) -> str:
        return "Buy 2 Get 1 Free"

class Order:
    def __init__(self, product: str, price: float, qty: int = 1) -> None:
        self.product = product
        self.price = price
        self.qty = qty
        self.discount_strategy: DiscountStrategy = NoDiscount()
    
    def set_discount(self, strategy: DiscountStrategy) -> None:
        self.discount_strategy = strategy
    
    @property
    def subtotal(self) -> float:
        return self.price * self.qty
    
    @property
    def final_price(self) -> float:
        return self.discount_strategy.calculate(self.subtotal)
    
    def print_receipt(self) -> None:
        print(f"  {self.product} x{self.qty}")
        print(f"  Original: {self.subtotal:.2f} THB")
        print(f"  Discount: {self.discount_strategy.description()}")
        print(f"  Final:    {self.final_price:.2f} THB")

print("\n=== Discount Strategies ===")
order = Order("Python Course", 990.0)
order.print_receipt()

order.set_discount(PercentageDiscount(20))
order.print_receipt()

order2 = Order("Coffee", 150.0, qty=3)
order2.set_discount(BuyTwoGetOneFree())
order2.print_receipt()

order3 = Order("Book", 450.0)
order3.set_discount(FlatDiscount(100))
order3.print_receipt()
```

---

## 7. Builder Pattern

```python
# Builder: สร้าง complex objects ทีละขั้น
# แยก construction จาก representation

class QueryBuilder:
    """SQL Query Builder"""
    
    def __init__(self) -> None:
        self._table: str = ""
        self._columns: List[str] = []
        self._conditions: List[str] = []
        self._order_by: Optional[str] = None
        self._order_dir: str = "ASC"
        self._limit: Optional[int] = None
        self._offset: int = 0
        self._joins: List[str] = []
    
    def from_table(self, table: str) -> 'QueryBuilder':
        self._table = table
        return self  # method chaining
    
    def select(self, *columns: str) -> 'QueryBuilder':
        self._columns.extend(columns)
        return self
    
    def where(self, condition: str) -> 'QueryBuilder':
        self._conditions.append(condition)
        return self
    
    def join(self, table: str, on: str) -> 'QueryBuilder':
        self._joins.append(f"JOIN {table} ON {on}")
        return self
    
    def order_by(self, column: str, direction: str = "ASC") -> 'QueryBuilder':
        self._order_by = column
        self._order_dir = direction
        return self
    
    def limit(self, n: int) -> 'QueryBuilder':
        self._limit = n
        return self
    
    def offset(self, n: int) -> 'QueryBuilder':
        self._offset = n
        return self
    
    def build(self) -> str:
        if not self._table:
            raise ValueError("Table name is required")
        
        cols = ", ".join(self._columns) if self._columns else "*"
        query = f"SELECT {cols} FROM {self._table}"
        
        for join in self._joins:
            query += f"\n{join}"
        
        if self._conditions:
            query += "\nWHERE " + " AND ".join(self._conditions)
        
        if self._order_by:
            query += f"\nORDER BY {self._order_by} {self._order_dir}"
        
        if self._limit:
            query += f"\nLIMIT {self._limit}"
        
        if self._offset:
            query += f"\nOFFSET {self._offset}"
        
        return query

# ใช้งาน Builder
print("=== Query Builder ===")

# Simple query
q1 = (QueryBuilder()
    .from_table("users")
    .select("id", "name", "email")
    .where("is_active = 1")
    .order_by("name")
    .limit(10)
    .build())
print(q1)
print()

# Complex query
q2 = (QueryBuilder()
    .from_table("orders")
    .select("o.id", "u.name", "o.total", "o.status")
    .join("users u", "u.id = o.user_id")
    .join("products p", "p.id = o.product_id")
    .where("o.status = 'pending'")
    .where("o.total > 1000")
    .order_by("o.created_at", "DESC")
    .limit(20)
    .offset(40)
    .build())
print(q2)
```

---

## 8. Command Pattern

```python
# Command: encapsulate operation เป็น object
# ใช้เมื่อ: undo/redo, queue operations, logging

from abc import ABC, abstractmethod
from typing import List

class Command(ABC):
    @abstractmethod
    def execute(self) -> None: ...
    
    @abstractmethod
    def undo(self) -> None: ...

class TextEditor:
    def __init__(self) -> None:
        self.text = ""
    
    def __str__(self) -> str:
        return f"Text: '{self.text}'"

# Concrete Commands
class TypeCommand(Command):
    def __init__(self, editor: TextEditor, text: str) -> None:
        self.editor = editor
        self.text = text
        self._old_text = ""
    
    def execute(self) -> None:
        self._old_text = self.editor.text
        self.editor.text += self.text
    
    def undo(self) -> None:
        self.editor.text = self._old_text

class DeleteCommand(Command):
    def __init__(self, editor: TextEditor, n_chars: int) -> None:
        self.editor = editor
        self.n_chars = n_chars
        self._deleted = ""
    
    def execute(self) -> None:
        self._deleted = self.editor.text[-self.n_chars:]
        self.editor.text = self.editor.text[:-self.n_chars]
    
    def undo(self) -> None:
        self.editor.text += self._deleted

class CommandHistory:
    def __init__(self) -> None:
        self._history: List[Command] = []
        self._redo_stack: List[Command] = []
    
    def execute(self, cmd: Command) -> None:
        cmd.execute()
        self._history.append(cmd)
        self._redo_stack.clear()  # clear redo stack
    
    def undo(self) -> bool:
        if not self._history:
            return False
        cmd = self._history.pop()
        cmd.undo()
        self._redo_stack.append(cmd)
        return True
    
    def redo(self) -> bool:
        if not self._redo_stack:
            return False
        cmd = self._redo_stack.pop()
        cmd.execute()
        self._history.append(cmd)
        return True

print("=== Text Editor with Undo/Redo ===")
editor = TextEditor()
history = CommandHistory()

history.execute(TypeCommand(editor, "Hello"))
print(editor)  # Text: 'Hello'

history.execute(TypeCommand(editor, ", World"))
print(editor)  # Text: 'Hello, World'

history.execute(TypeCommand(editor, "!"))
print(editor)  # Text: 'Hello, World!'

print("\n-- Undo x2 --")
history.undo()
print(editor)  # Text: 'Hello, World'

history.undo()
print(editor)  # Text: 'Hello'

print("\n-- Redo --")
history.redo()
print(editor)  # Text: 'Hello, World'

print("\n-- Delete 6 chars --")
history.execute(DeleteCommand(editor, 6))
print(editor)  # Text: 'Hello'

history.undo()
print(editor)  # Text: 'Hello, World'
```

---

## 9. สรุป Part 030

✅ **Singleton** - instance เดียวตลอด program (config, logger, DB connection)  
✅ **Factory** - สร้าง objects โดยไม่ระบุ class ตรงๆ (notifications, themes)  
✅ **Observer** - notify observers เมื่อ state เปลี่ยน (events, pub/sub)  
✅ **Decorator Pattern** - เพิ่ม behavior โดยไม่แก้ class เดิม (cache, logging)  
✅ **Strategy** - swap algorithms ที่ runtime (sort, discount)  
✅ **Builder** - สร้าง complex objects ทีละขั้น (query builder)  
✅ **Command** - encapsulate operations (undo/redo)  

---

## ➡️ ถัดไป: Part 031 - Decorators และ Descriptors

*Part 030/100+ | Python Course - Beginner to World-Class*
