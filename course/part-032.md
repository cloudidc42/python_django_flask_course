# Part 032: Concurrency - Threading
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Concurrency vs Parallelism
- ใช้ threading module สร้าง Threads
- Thread Class, start(), join(), daemon threads
- Locks, RLock สำหรับ thread safety
- Semaphore สำหรับ limiting concurrent access
- Queue สำหรับ thread communication
- concurrent.futures ThreadPoolExecutor
- เข้าใจ GIL และเมื่อไหร่ควรใช้ threads

---

## 1. Concurrency vs Parallelism

```python
# Concurrency = จัดการหลายงานพร้อมกัน (switching ระหว่าง tasks)
# Parallelism = ทำหลายงานพร้อมกันจริงๆ (หลาย CPU cores)

# Python มี GIL (Global Interpreter Lock):
# - ทำให้ Python threads ทำงานได้ทีละ thread เท่านั้น
# - Threading ดีสำหรับ I/O-bound tasks (รอ network, file, DB)
# - Threading ไม่ได้เร็วขึ้นสำหรับ CPU-bound tasks
# - สำหรับ CPU-bound ใช้ multiprocessing (Part 033)

# I/O-bound: ใช้ threading หรือ asyncio
# ตัวอย่าง: download files, API calls, database queries

# CPU-bound: ใช้ multiprocessing
# ตัวอย่าง: image processing, calculations, data analysis

import time

def task_sequential(name: str, duration: float) -> None:
    """Simulate I/O operation (เช่น HTTP request)"""
    print(f"  [{name}] Started")
    time.sleep(duration)
    print(f"  [{name}] Done after {duration}s")

# Sequential: รอให้เสร็จทีละงาน
print("=== Sequential ===")
start = time.time()
task_sequential("Task A", 1.0)
task_sequential("Task B", 1.0)
task_sequential("Task C", 1.0)
print(f"Total time: {time.time() - start:.2f}s")
# Total time: ~3.00s
```

---

## 2. Thread พื้นฐาน

```python
import threading
import time

def download_file(filename: str, duration: float) -> None:
    """Simulate file download"""
    print(f"  Downloading {filename}...")
    time.sleep(duration)
    print(f"  ✅ {filename} downloaded!")

# สร้าง threads
t1 = threading.Thread(target=download_file, args=("file1.zip", 2.0))
t2 = threading.Thread(target=download_file, args=("file2.zip", 1.5))
t3 = threading.Thread(target=download_file, args=("file3.zip", 1.0))

print("=== Threading ===")
start = time.time()

# Start threads
t1.start()
t2.start()
t3.start()

# Wait for all threads to finish
t1.join()
t2.join()
t3.join()

elapsed = time.time() - start
print(f"Total time: {elapsed:.2f}s")
# Total time: ~2.00s (เร็วกว่า sequential ~3x)

# สร้าง Thread ด้วย class
class WorkerThread(threading.Thread):
    def __init__(self, name: str, work_items: list) -> None:
        super().__init__(name=name, daemon=True)
        self.work_items = work_items
        self.results = []
    
    def run(self) -> None:
        """run() ถูกเรียกเมื่อ thread start"""
        print(f"  [{self.name}] Starting with {len(self.work_items)} items")
        for item in self.work_items:
            result = self.process(item)
            self.results.append(result)
            time.sleep(0.1)
        print(f"  [{self.name}] Done! Processed {len(self.results)} items")
    
    def process(self, item: int) -> int:
        return item * item  # Square

# ใช้งาน
items1 = list(range(1, 6))   # [1, 2, 3, 4, 5]
items2 = list(range(6, 11))  # [6, 7, 8, 9, 10]

worker1 = WorkerThread("Worker-1", items1)
worker2 = WorkerThread("Worker-2", items2)

worker1.start()
worker2.start()

worker1.join()
worker2.join()

print(f"Worker1 results: {worker1.results}")
print(f"Worker2 results: {worker2.results}")
# Worker1 results: [1, 4, 9, 16, 25]
# Worker2 results: [36, 49, 64, 81, 100]

# Daemon Thread: thread ที่จะถูก kill เมื่อ main thread จบ
def background_monitor() -> None:
    while True:
        print("  [Monitor] System healthy...")
        time.sleep(1)

monitor = threading.Thread(target=background_monitor, daemon=True)
monitor.start()

# Thread info
print(f"\nActive threads: {threading.active_count()}")
print(f"Current thread: {threading.current_thread().name}")
```

---

## 3. Thread Safety: Locks

```python
import threading
import time

# ปัญหา: Race Condition
class BankAccountUnsafe:
    def __init__(self, balance: float) -> None:
        self.balance = balance
    
    def deposit(self, amount: float) -> None:
        # ⚠️ Not thread-safe!
        current = self.balance
        time.sleep(0.001)  # Simulate processing
        self.balance = current + amount
    
    def withdraw(self, amount: float) -> bool:
        if self.balance >= amount:
            current = self.balance
            time.sleep(0.001)
            self.balance = current - amount
            return True
        return False

# ทดสอบ race condition
account = BankAccountUnsafe(1000.0)
threads = []

for _ in range(10):
    t = threading.Thread(target=account.deposit, args=(100.0,))
    threads.append(t)

for t in threads:
    t.start()
for t in threads:
    t.join()

print(f"Expected: 2000.0, Got: {account.balance}")
# อาจได้ค่าไม่ถูกต้อง เช่น 1200.0

# ✅ แก้ด้วย Lock
class BankAccount:
    def __init__(self, owner: str, balance: float) -> None:
        self.owner = owner
        self._balance = balance
        self._lock = threading.Lock()
        self._transactions = []
    
    def deposit(self, amount: float, description: str = "") -> None:
        with self._lock:  # Only one thread at a time
            if amount <= 0:
                raise ValueError("Amount must be positive")
            old_balance = self._balance
            self._balance += amount
            self._transactions.append({
                "type": "deposit",
                "amount": amount,
                "balance": self._balance,
                "desc": description
            })
            print(f"  [{self.owner}] +{amount:.2f} | Balance: {self._balance:.2f}")
    
    def withdraw(self, amount: float, description: str = "") -> bool:
        with self._lock:
            if amount <= 0:
                raise ValueError("Amount must be positive")
            if self._balance < amount:
                print(f"  [{self.owner}] Insufficient funds! Need {amount:.2f}, have {self._balance:.2f}")
                return False
            self._balance -= amount
            self._transactions.append({
                "type": "withdrawal",
                "amount": amount,
                "balance": self._balance,
                "desc": description
            })
            print(f"  [{self.owner}] -{amount:.2f} | Balance: {self._balance:.2f}")
            return True
    
    @property
    def balance(self) -> float:
        with self._lock:
            return self._balance

# ทดสอบ thread-safe
account = BankAccount("Alice", 1000.0)
threads = []

for i in range(5):
    t = threading.Thread(target=account.deposit, args=(100.0, f"Deposit {i+1}"))
    threads.append(t)

for i in range(3):
    t = threading.Thread(target=account.withdraw, args=(150.0, f"Withdraw {i+1}"))
    threads.append(t)

import random
random.shuffle(threads)  # สุ่ม order

for t in threads:
    t.start()
for t in threads:
    t.join()

print(f"\nFinal balance: {account.balance:.2f}")
print(f"Transactions: {len(account._transactions)}")

# RLock: Re-entrant Lock (สำหรับ recursive locking)
class RecursiveOperation:
    def __init__(self) -> None:
        self._lock = threading.RLock()  # ใช้ RLock ไม่ใช่ Lock
        self.value = 0
    
    def increment(self) -> None:
        with self._lock:
            self.value += 1
            self.double_if_even()  # เรียก method อื่นที่ใช้ lock เดิม
    
    def double_if_even(self) -> None:
        with self._lock:  # RLock อนุญาตให้ acquire ซ้ำจาก thread เดิม
            if self.value % 2 == 0:
                self.value *= 2
```

---

## 4. Semaphore: Limit Concurrent Access

```python
import threading
import time
import random

# Semaphore: จำกัดจำนวน threads ที่ access resource พร้อมกัน
# ใช้เมื่อ: limit DB connections, API rate limiting

class DatabasePool:
    """Simulate database connection pool"""
    
    def __init__(self, max_connections: int = 3) -> None:
        self.max_connections = max_connections
        self._semaphore = threading.Semaphore(max_connections)
        self._active = 0
        self._lock = threading.Lock()
    
    def get_connection(self) -> 'DatabasePool':
        return self
    
    def __enter__(self):
        self._semaphore.acquire()
        with self._lock:
            self._active += 1
            print(f"  🔌 Connection acquired (active: {self._active}/{self.max_connections})")
        return self
    
    def __exit__(self, *args):
        with self._lock:
            self._active -= 1
            print(f"  🔌 Connection released (active: {self._active}/{self.max_connections})")
        self._semaphore.release()
    
    def query(self, sql: str) -> list:
        time.sleep(random.uniform(0.5, 1.5))  # Simulate query
        return [{"result": "data"}]

def process_request(pool: DatabasePool, request_id: int) -> None:
    """Simulate HTTP request handling"""
    print(f"  Request {request_id}: waiting for connection...")
    with pool:
        print(f"  Request {request_id}: executing query...")
        result = pool.query(f"SELECT * WHERE id={request_id}")
        print(f"  Request {request_id}: completed!")

print("=== Connection Pool Demo ===")
pool = DatabasePool(max_connections=3)

# 10 concurrent requests แต่ pool มีแค่ 3 connections
threads = [
    threading.Thread(target=process_request, args=(pool, i))
    for i in range(1, 8)
]

start = time.time()
for t in threads:
    t.start()
for t in threads:
    t.join()

print(f"All requests completed in {time.time() - start:.2f}s")

# Bounded Semaphore: เหมือน Semaphore แต่ panic ถ้า release มากกว่า acquire
bounded = threading.BoundedSemaphore(5)
bounded.acquire()
bounded.release()
# bounded.release()  # ValueError: Semaphore released too many times
```

---

## 5. Queue: Thread Communication

```python
import threading
import queue
import time
import random
from typing import Optional

# Queue: thread-safe FIFO queue สำหรับ producer-consumer pattern

def producer(q: queue.Queue, num_items: int, producer_id: int) -> None:
    """สร้าง work items และใส่ใน queue"""
    for i in range(num_items):
        item = f"item-{producer_id}-{i}"
        q.put(item)
        print(f"  [Producer {producer_id}] Produced: {item}")
        time.sleep(random.uniform(0.1, 0.3))
    
    # บอก consumers ว่า producer นี้เสร็จแล้ว
    print(f"  [Producer {producer_id}] Done!")

def consumer(q: queue.Queue, consumer_id: int) -> None:
    """รับ work items จาก queue และ process"""
    processed = 0
    while True:
        try:
            # timeout=1: รอสูงสุด 1 วินาที
            item = q.get(timeout=1.0)
            print(f"  [Consumer {consumer_id}] Processing: {item}")
            time.sleep(random.uniform(0.2, 0.5))  # Simulate work
            q.task_done()  # บอกว่า item นี้เสร็จแล้ว
            processed += 1
        except queue.Empty:
            print(f"  [Consumer {consumer_id}] No more items. Processed: {processed}")
            break

print("=== Producer-Consumer ===")
work_queue = queue.Queue(maxsize=5)  # maxsize=5: จะ block ถ้า queue เต็ม

# 2 producers, 3 consumers
producers = [
    threading.Thread(target=producer, args=(work_queue, 3, i+1))
    for i in range(2)
]
consumers = [
    threading.Thread(target=consumer, args=(work_queue, i+1))
    for i in range(3)
]

# Start all
for p in producers:
    p.start()
for c in consumers:
    c.start()

# Wait for producers
for p in producers:
    p.join()

# Wait for all items to be processed
work_queue.join()
print("All items processed!")

# Priority Queue
pq = queue.PriorityQueue()
pq.put((3, "low priority task"))
pq.put((1, "high priority task"))
pq.put((2, "medium priority task"))

while not pq.empty():
    priority, task = pq.get()
    print(f"  [{priority}] {task}")
# [1] high priority task
# [2] medium priority task
# [3] low priority task

# Thread-safe results collection
class ThreadSafeList:
    def __init__(self):
        self._items = []
        self._lock = threading.Lock()
    
    def append(self, item) -> None:
        with self._lock:
            self._items.append(item)
    
    def get_all(self) -> list:
        with self._lock:
            return self._items.copy()

results = ThreadSafeList()

def fetch_data(url: str) -> None:
    """Simulate fetch"""
    time.sleep(random.uniform(0.1, 0.5))
    results.append({"url": url, "status": 200})

urls = [f"https://api.example.com/{i}" for i in range(5)]
threads = [threading.Thread(target=fetch_data, args=(url,)) for url in urls]
for t in threads:
    t.start()
for t in threads:
    t.join()

print(f"Fetched {len(results.get_all())} URLs")
```

---

## 6. concurrent.futures ThreadPoolExecutor

```python
from concurrent.futures import ThreadPoolExecutor, as_completed, Future
import time
import random
from typing import List, Dict

def fetch_user(user_id: int) -> Dict:
    """Simulate API call"""
    time.sleep(random.uniform(0.5, 1.5))
    return {
        "id": user_id,
        "name": f"User {user_id}",
        "email": f"user{user_id}@example.com",
        "score": random.randint(1, 100)
    }

def process_image(filename: str) -> str:
    """Simulate image processing"""
    time.sleep(random.uniform(0.3, 0.8))
    return f"processed_{filename}"

print("=== ThreadPoolExecutor ===")

# map() - ง่ายสุด
user_ids = list(range(1, 11))

start = time.time()
with ThreadPoolExecutor(max_workers=5) as executor:
    # map จะ block จนทุกงานเสร็จ
    users = list(executor.map(fetch_user, user_ids))

print(f"Fetched {len(users)} users in {time.time() - start:.2f}s")
print(f"Top scorer: {max(users, key=lambda u: u['score'])['name']}")

# submit() - ควบคุมได้มากกว่า
print("\n--- submit() with as_completed ---")
with ThreadPoolExecutor(max_workers=4) as executor:
    # Submit tasks ทั้งหมด
    futures = {
        executor.submit(fetch_user, uid): uid
        for uid in range(1, 6)
    }
    
    # Process results as they complete (ไม่รอตาม order)
    for future in as_completed(futures):
        user_id = futures[future]
        try:
            user = future.result()
            print(f"  ✅ User {user['id']}: {user['name']} (score={user['score']})")
        except Exception as e:
            print(f"  ❌ User {user_id} failed: {e}")

# Future object
print("\n--- Future operations ---")
with ThreadPoolExecutor(max_workers=2) as executor:
    future1: Future = executor.submit(fetch_user, 100)
    future2: Future = executor.submit(fetch_user, 200)
    
    print(f"  future1 done: {future1.done()}")
    
    result1 = future1.result(timeout=5.0)  # รอสูงสุด 5 วินาที
    result2 = future2.result()
    
    print(f"  User 100: {result1['name']}")
    print(f"  User 200: {result2['name']}")
    print(f"  future1 done: {future1.done()}")

# Batch processing with progress
print("\n--- Batch image processing ---")
filenames = [f"photo_{i:03d}.jpg" for i in range(1, 9)]
processed = []

with ThreadPoolExecutor(max_workers=3) as executor:
    futures_map = {executor.submit(process_image, fn): fn for fn in filenames}
    total = len(futures_map)
    done = 0
    
    for future in as_completed(futures_map):
        result = future.result()
        processed.append(result)
        done += 1
        print(f"  Progress: {done}/{total} - {result}")

print(f"Processed: {len(processed)} images")
```

---

## 7. Event และ Condition

```python
import threading
import time

# Event: สัญญาณง่ายๆ ระหว่าง threads
class DataProcessor:
    def __init__(self):
        self._ready_event = threading.Event()
        self._data = None
    
    def load_data(self) -> None:
        """Thread นี้โหลดข้อมูล"""
        print("  [Loader] Loading data...")
        time.sleep(2.0)  # Simulate loading
        self._data = {"records": list(range(100))}
        print("  [Loader] Data loaded!")
        self._ready_event.set()  # Signal ว่าพร้อมแล้ว
    
    def process_data(self) -> None:
        """Thread นี้รอข้อมูลก่อน process"""
        print("  [Processor] Waiting for data...")
        self._ready_event.wait()  # Block จน event ถูก set
        print(f"  [Processor] Processing {len(self._data['records'])} records...")
        time.sleep(0.5)
        print("  [Processor] Done!")

print("=== Event Demo ===")
processor = DataProcessor()
loader_thread = threading.Thread(target=processor.load_data)
worker_thread = threading.Thread(target=processor.process_data)

worker_thread.start()
loader_thread.start()

loader_thread.join()
worker_thread.join()

# Condition: Event ที่ซับซ้อนขึ้น
class BoundedBuffer:
    """Thread-safe bounded buffer"""
    
    def __init__(self, capacity: int) -> None:
        self.capacity = capacity
        self.buffer = []
        self.condition = threading.Condition()
    
    def put(self, item) -> None:
        with self.condition:
            while len(self.buffer) >= self.capacity:
                print(f"  Buffer full, producer waiting...")
                self.condition.wait()
            self.buffer.append(item)
            print(f"  Put {item} | Buffer: {self.buffer}")
            self.condition.notify_all()
    
    def get(self):
        with self.condition:
            while not self.buffer:
                print(f"  Buffer empty, consumer waiting...")
                self.condition.wait()
            item = self.buffer.pop(0)
            print(f"  Got {item} | Buffer: {self.buffer}")
            self.condition.notify_all()
            return item

buffer = BoundedBuffer(capacity=3)

def produce(buf, items):
    for item in items:
        buf.put(item)
        time.sleep(0.2)

def consume(buf, count):
    for _ in range(count):
        item = buf.get()
        time.sleep(0.5)

print("\n=== Bounded Buffer ===")
p = threading.Thread(target=produce, args=(buffer, [1,2,3,4,5,6]))
c = threading.Thread(target=consume, args=(buffer, 6))

p.start()
c.start()
p.join()
c.join()
```

---

## 8. GIL และเมื่อไหร่ควรใช้ Threads

```python
import threading
import time

# GIL = Global Interpreter Lock
# - CPython ใช้ GIL ป้องกัน Python objects จาก concurrent modification
# - ทำให้ Python threads ทำงาน "one at a time" ใน CPU
# - ยกเว้นเมื่อ thread ทำ I/O (release GIL ให้ thread อื่น)

# ✅ Threading เหมาะสำหรับ I/O-bound:
# - HTTP requests
# - File I/O
# - Database queries
# - Sleep

# ❌ Threading ไม่เหมาะสำหรับ CPU-bound:
# - Number crunching
# - Image processing
# - Encryption

# ทดสอบ CPU-bound (threading ไม่ช่วย)
def cpu_intensive(n: int) -> int:
    """คำนวณ sum of squares"""
    return sum(i*i for i in range(n))

# Sequential
start = time.time()
results = [cpu_intensive(1_000_000) for _ in range(4)]
sequential_time = time.time() - start
print(f"Sequential: {sequential_time:.3f}s")

# Threading (ไม่เร็วกว่าเพราะ GIL)
start = time.time()
threads = [threading.Thread(target=cpu_intensive, args=(1_000_000,)) for _ in range(4)]
for t in threads:
    t.start()
for t in threads:
    t.join()
threading_time = time.time() - start
print(f"Threading:  {threading_time:.3f}s (probably similar or slower!)")

# ทดสอบ I/O-bound (threading ช่วยได้มาก)
def io_task(duration: float) -> None:
    time.sleep(duration)

# Sequential
start = time.time()
for _ in range(4):
    io_task(0.5)
sequential_time = time.time() - start
print(f"\nI/O Sequential: {sequential_time:.2f}s")

# Threading
start = time.time()
threads = [threading.Thread(target=io_task, args=(0.5,)) for _ in range(4)]
for t in threads:
    t.start()
for t in threads:
    t.join()
threading_time = time.time() - start
print(f"I/O Threading:  {threading_time:.2f}s (much faster!)")

# สรุป: เมื่อไหร่ควรใช้ threading?
use_cases = {
    "✅ Web scraping": "รอ HTTP responses",
    "✅ API calls": "รอ server responses",
    "✅ File upload/download": "รอ I/O",
    "✅ Database queries": "รอ DB response",
    "✅ Background tasks": "Email, notifications",
    "❌ Image processing": "ใช้ multiprocessing แทน",
    "❌ Machine learning": "ใช้ multiprocessing หรือ GPU",
    "❌ Data transformation": "ใช้ multiprocessing แทน",
}

print("\n=== When to use Threading ===")
for case, reason in use_cases.items():
    print(f"  {case}: {reason}")
```

---

## 9. ตัวอย่างจริง: Concurrent Web Scraper

```python
import threading
import time
import random
from concurrent.futures import ThreadPoolExecutor, as_completed
from typing import Dict, List, Optional
from dataclasses import dataclass, field
from queue import Queue

@dataclass
class ScrapeResult:
    url: str
    status: int
    title: str
    content_length: int
    elapsed: float
    error: Optional[str] = None

def scrape_url(url: str) -> ScrapeResult:
    """Simulate web scraping"""
    start = time.time()
    
    # Simulate network request
    time.sleep(random.uniform(0.3, 2.0))
    
    # Simulate occasional errors
    if random.random() < 0.1:  # 10% error rate
        return ScrapeResult(
            url=url, status=500,
            title="", content_length=0,
            elapsed=time.time()-start,
            error="Server Error"
        )
    
    return ScrapeResult(
        url=url,
        status=200,
        title=f"Page: {url.split('/')[-1]}",
        content_length=random.randint(1000, 50000),
        elapsed=time.time()-start
    )

class WebScraper:
    def __init__(self, max_workers: int = 5) -> None:
        self.max_workers = max_workers
        self._results: List[ScrapeResult] = []
        self._lock = threading.Lock()
        self._stats = {"success": 0, "error": 0}
    
    def scrape_all(self, urls: List[str]) -> List[ScrapeResult]:
        print(f"Scraping {len(urls)} URLs with {self.max_workers} workers...")
        
        with ThreadPoolExecutor(max_workers=self.max_workers) as executor:
            futures = {executor.submit(scrape_url, url): url for url in urls}
            
            for future in as_completed(futures):
                result = future.result()
                with self._lock:
                    self._results.append(result)
                    if result.error:
                        self._stats["error"] += 1
                        print(f"  ❌ {result.url}: {result.error}")
                    else:
                        self._stats["success"] += 1
                        print(f"  ✅ {result.url}: {result.content_length} bytes in {result.elapsed:.2f}s")
        
        return self._results
    
    def report(self) -> None:
        if not self._results:
            return
        
        success = [r for r in self._results if not r.error]
        errors = [r for r in self._results if r.error]
        
        print(f"\n=== Scraping Report ===")
        print(f"Total: {len(self._results)}")
        print(f"Success: {len(success)} ({len(success)/len(self._results)*100:.1f}%)")
        print(f"Errors: {len(errors)}")
        
        if success:
            avg_time = sum(r.elapsed for r in success) / len(success)
            total_bytes = sum(r.content_length for r in success)
            print(f"Avg response time: {avg_time:.2f}s")
            print(f"Total content: {total_bytes:,} bytes")
        
        print("\nTop 3 largest pages:")
        for r in sorted(success, key=lambda x: x.content_length, reverse=True)[:3]:
            print(f"  {r.url}: {r.content_length:,} bytes")

# ใช้งาน
urls = [
    f"https://example.com/page/{i}"
    for i in range(1, 16)
]

scraper = WebScraper(max_workers=5)
start = time.time()
results = scraper.scrape_all(urls)
total_time = time.time() - start

print(f"\nCompleted in {total_time:.2f}s")
scraper.report()
```

---

## 10. สรุป Part 032

✅ **threading.Thread** สร้างและรัน threads  
✅ **start() + join()** เพื่อ launch และรอ threads  
✅ **Daemon threads** จบเมื่อ main thread จบ  
✅ **Lock** ป้องกัน race conditions (thread safety)  
✅ **Semaphore** จำกัด concurrent access  
✅ **Queue** สำหรับ producer-consumer pattern  
✅ **ThreadPoolExecutor** จัดการ pool of threads  
✅ **GIL** ทำให้ threading ดีแค่ I/O-bound tasks  
✅ **I/O-bound**: threading ✅ | **CPU-bound**: multiprocessing ✅  

---

## ➡️ ถัดไป: Part 033 - Multiprocessing

*Part 032/100+ | Python Course - Beginner to World-Class*
