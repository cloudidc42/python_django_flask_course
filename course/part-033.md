# Part 033: Multiprocessing
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจความต่างของ multiprocessing กับ threading
- ใช้ Process class สร้าง subprocesses
- Pool สำหรับ parallel processing
- Shared memory: Value, Array, Manager
- Queue และ Pipe สำหรับ inter-process communication
- concurrent.futures ProcessPoolExecutor
- เมื่อไหร่ควรใช้ multiprocessing

---

## 1. Multiprocessing vs Threading

```python
# Threading: หลาย threads ใน process เดียว
#   - ใช้ memory ร่วมกัน
#   - GIL จำกัดให้ CPU ทำงาน thread เดียวต่อครั้ง
#   - เหมาะกับ I/O-bound tasks
#
# Multiprocessing: หลาย processes แยกกัน
#   - แต่ละ process มี memory ของตัวเอง
#   - ไม่มี GIL → ทำงานบน multiple CPU cores พร้อมกันได้
#   - เหมาะกับ CPU-bound tasks

import time
import multiprocessing as mp
import threading

def cpu_task(n: int) -> int:
    """CPU-intensive task"""
    return sum(i * i for i in range(n))

# Sequential
start = time.time()
results = [cpu_task(5_000_000) for _ in range(4)]
print(f"Sequential:      {time.time()-start:.2f}s")

# Threading (GIL: ไม่เร็วขึ้นมาก)
start = time.time()
threads = [threading.Thread(target=cpu_task, args=(5_000_000,)) for _ in range(4)]
for t in threads: t.start()
for t in threads: t.join()
print(f"Threading:       {time.time()-start:.2f}s  (limited by GIL)")

# Multiprocessing (True parallelism)
start = time.time()
with mp.Pool(processes=4) as pool:
    results = pool.map(cpu_task, [5_000_000] * 4)
print(f"Multiprocessing: {time.time()-start:.2f}s  (uses all CPU cores!)")

# ดูจำนวน CPU cores
print(f"\nCPU cores: {mp.cpu_count()}")
```

---

## 2. Process Class พื้นฐาน

```python
import multiprocessing as mp
import os
import time

def worker_function(name: str, duration: float) -> None:
    """Function ที่รันใน subprocess"""
    pid = os.getpid()
    print(f"  [{name}] PID={pid} started")
    time.sleep(duration)
    print(f"  [{name}] PID={pid} done after {duration}s")

def compute_square(x: int) -> None:
    print(f"  {x}^2 = {x**2} (PID={os.getpid()})")

# สร้าง Process
print("=== Basic Process ===")
print(f"Main process PID: {os.getpid()}")

p1 = mp.Process(target=worker_function, args=("Worker-A", 1.0))
p2 = mp.Process(target=worker_function, args=("Worker-B", 1.5))
p3 = mp.Process(target=worker_function, args=("Worker-C", 0.5))

start = time.time()
p1.start()
p2.start()
p3.start()

p1.join()
p2.join()
p3.join()

print(f"Total: {time.time()-start:.2f}s")

# Process attributes
p = mp.Process(target=compute_square, args=(10,))
print(f"\nBefore start - alive: {p.is_alive()}")
p.start()
print(f"After start  - alive: {p.is_alive()}, PID: {p.pid}")
p.join()
print(f"After join   - alive: {p.is_alive()}, exitcode: {p.exitcode}")

# Process subclass
class DataWorker(mp.Process):
    def __init__(self, data: list, worker_id: int) -> None:
        super().__init__()
        self.data = data
        self.worker_id = worker_id
    
    def run(self) -> None:
        """run() ถูกเรียกเมื่อ process start"""
        print(f"  [Worker {self.worker_id}] Processing {len(self.data)} items on PID={os.getpid()}")
        result = sum(x**2 for x in self.data)
        print(f"  [Worker {self.worker_id}] Result: {result}")

chunks = [[1,2,3,4,5], [6,7,8,9,10], [11,12,13,14,15]]
workers = [DataWorker(chunk, i+1) for i, chunk in enumerate(chunks)]

for w in workers:
    w.start()
for w in workers:
    w.join()

# terminate(): kill process ทันที
import signal

def long_running() -> None:
    print(f"  Long process started (PID={os.getpid()})")
    time.sleep(100)

p = mp.Process(target=long_running)
p.start()
time.sleep(0.5)
p.terminate()
p.join()
print(f"Process terminated. exitcode: {p.exitcode}")
# exitcode: -15 (SIGTERM)
```

---

## 3. Pool: Parallel Processing

```python
import multiprocessing as mp
import time
from typing import List, Tuple
import math

def is_prime(n: int) -> bool:
    """ตรวจสอบว่าเป็นจำนวนเฉพาะ"""
    if n < 2:
        return False
    if n == 2:
        return True
    if n % 2 == 0:
        return False
    for i in range(3, int(math.sqrt(n)) + 1, 2):
        if n % i == 0:
            return False
    return True

def process_batch(numbers: List[int]) -> List[int]:
    """หาจำนวนเฉพาะใน batch"""
    return [n for n in numbers if is_prime(n)]

# Pool.map() - parallel map
numbers = list(range(1, 100_001))

start = time.time()
primes_seq = [n for n in numbers if is_prime(n)]
print(f"Sequential: {time.time()-start:.2f}s - found {len(primes_seq)} primes")

start = time.time()
with mp.Pool(processes=mp.cpu_count()) as pool:
    # แบ่ง numbers เป็น chunks แล้วส่งให้แต่ละ process
    chunk_size = len(numbers) // mp.cpu_count()
    chunks = [numbers[i:i+chunk_size] for i in range(0, len(numbers), chunk_size)]
    
    results = pool.map(process_batch, chunks)
    primes_par = [p for batch in results for p in batch]

print(f"Parallel:   {time.time()-start:.2f}s - found {len(primes_par)} primes")

# Pool.starmap() - map กับหลาย arguments
def power(base: int, exp: int) -> int:
    return base ** exp

args = [(2, 10), (3, 5), (4, 4), (5, 3), (10, 6)]

with mp.Pool(processes=4) as pool:
    results = pool.starmap(power, args)

print("\nPower calculations:")
for (base, exp), result in zip(args, results):
    print(f"  {base}^{exp} = {result}")

# Pool.imap() - lazy evaluation (ดีสำหรับ large datasets)
def slow_square(n: int) -> int:
    time.sleep(0.01)
    return n ** 2

print("\nimap results:")
with mp.Pool(processes=4) as pool:
    for result in pool.imap(slow_square, range(1, 6)):
        print(f"  {result}", end=" ", flush=True)
print()

# Pool.apply_async() - asynchronous
def fetch_data(url_id: int) -> dict:
    time.sleep(0.2)
    return {"id": url_id, "data": f"content_{url_id}"}

print("\napply_async results:")
with mp.Pool(processes=4) as pool:
    async_results = [pool.apply_async(fetch_data, (i,)) for i in range(1, 6)]
    
    for ar in async_results:
        result = ar.get(timeout=5)
        print(f"  {result}")

# Pool context manager ดูแล cleanup อัตโนมัติ
# ไม่ต้องเรียก pool.close() และ pool.join() เอง
```

---

## 4. Shared Memory: Value และ Array

```python
import multiprocessing as mp
import time

# ปัญหา: processes แยก memory → ไม่ share ตัวแปรกัน
def bad_increment(counter, n: int) -> None:
    """ไม่ทำงานเพราะ counter ถูก copy"""
    for _ in range(n):
        counter += 1
    print(f"  Local counter: {counter}")

# ทดสอบ
x = 0
processes = [mp.Process(target=bad_increment, args=(x, 100)) for _ in range(4)]
for p in processes: p.start()
for p in processes: p.join()
print(f"Main counter (wrong): {x}")  # ยังคงเป็น 0!

# ✅ Value: shared single value
counter = mp.Value('i', 0)  # 'i' = signed int, 0 = initial value
lock = mp.Lock()

def safe_increment(counter, lock, n: int) -> None:
    for _ in range(n):
        with lock:
            counter.value += 1

processes = [
    mp.Process(target=safe_increment, args=(counter, lock, 1000))
    for _ in range(4)
]
for p in processes: p.start()
for p in processes: p.join()
print(f"Shared counter (correct): {counter.value}")  # 4000

# Value types:
# 'b' = signed char
# 'i' = signed int
# 'l' = signed long
# 'f' = float
# 'd' = double

# ✅ Array: shared array
arr = mp.Array('d', [0.0] * 5)  # 'd' = double, size=5

def fill_array(arr, index: int, value: float) -> None:
    arr[index] = value * value

processes = [
    mp.Process(target=fill_array, args=(arr, i, float(i+1)))
    for i in range(5)
]
for p in processes: p.start()
for p in processes: p.join()

print(f"Shared array: {list(arr)}")  # [1.0, 4.0, 9.0, 16.0, 25.0]

# ✅ Manager: share complex objects
def update_dict(d: dict, key: str, value: int) -> None:
    d[key] = value

def append_list(lst: list, item) -> None:
    lst.append(item)

with mp.Manager() as manager:
    shared_dict = manager.dict()
    shared_list = manager.list()
    
    p1 = mp.Process(target=update_dict, args=(shared_dict, "a", 1))
    p2 = mp.Process(target=update_dict, args=(shared_dict, "b", 2))
    p3 = mp.Process(target=append_list, args=(shared_list, "hello"))
    p4 = mp.Process(target=append_list, args=(shared_list, "world"))
    
    for p in [p1, p2, p3, p4]: p.start()
    for p in [p1, p2, p3, p4]: p.join()
    
    print(f"Shared dict: {dict(shared_dict)}")
    print(f"Shared list: {list(shared_list)}")
```

---

## 5. Queue และ Pipe

```python
import multiprocessing as mp
import time
import random
from typing import Optional

# multiprocessing.Queue: thread-safe & process-safe
def producer(queue: mp.Queue, items: list, producer_id: int) -> None:
    for item in items:
        queue.put(item)
        print(f"  [Producer {producer_id}] Put: {item}")
        time.sleep(random.uniform(0.1, 0.3))
    queue.put(None)  # Sentinel value

def consumer(queue: mp.Queue, consumer_id: int) -> None:
    results = []
    while True:
        item = queue.get()
        if item is None:
            break
        result = item ** 2
        results.append(result)
        print(f"  [Consumer {consumer_id}] Got {item} -> {result}")
        time.sleep(0.2)
    print(f"  [Consumer {consumer_id}] Done: {results}")

print("=== Multiprocessing Queue ===")
q = mp.Queue(maxsize=5)

prod = mp.Process(target=producer, args=(q, list(range(1, 7)), 1))
cons = mp.Process(target=consumer, args=(q, 1))

prod.start()
cons.start()
prod.join()
cons.join()

# Pipe: คู่ connection ระหว่าง 2 processes
def sender(conn) -> None:
    messages = ["Hello", "World", "from", "subprocess"]
    for msg in messages:
        conn.send(msg)
        print(f"  [Sender] Sent: {msg}")
        time.sleep(0.3)
    conn.send(None)  # Done signal
    conn.close()

def receiver(conn) -> None:
    received = []
    while True:
        msg = conn.recv()
        if msg is None:
            break
        received.append(msg)
        print(f"  [Receiver] Got: {msg}")
    conn.close()
    print(f"  [Receiver] All received: {received}")

print("\n=== Pipe Demo ===")
parent_conn, child_conn = mp.Pipe()

p1 = mp.Process(target=sender, args=(child_conn,))
p2 = mp.Process(target=receiver, args=(parent_conn,))

p1.start()
p2.start()
p1.join()
p2.join()

# Duplex Pipe (bidirectional)
def bidirectional_worker(conn) -> None:
    """Ping-pong communication"""
    for _ in range(3):
        msg = conn.recv()
        print(f"  [Worker] Got: {msg}")
        conn.send(f"Reply to: {msg}")
    conn.close()

parent_conn, child_conn = mp.Pipe(duplex=True)

worker = mp.Process(target=bidirectional_worker, args=(child_conn,))
worker.start()

for i in range(3):
    parent_conn.send(f"Message {i+1}")
    reply = parent_conn.recv()
    print(f"  [Main] Got reply: {reply}")

parent_conn.close()
worker.join()
```

---

## 6. concurrent.futures ProcessPoolExecutor

```python
from concurrent.futures import ProcessPoolExecutor, as_completed
import time
import math
from typing import List, Tuple

def factorize(n: int) -> Tuple[int, List[int]]:
    """หา prime factors ของ n"""
    factors = []
    d = 2
    while d * d <= n:
        while n % d == 0:
            factors.append(d)
            n //= d
        d += 1
    if n > 1:
        factors.append(n)
    return (n if n > 1 else factors[-1] if factors else 1, factors)

def analyze_data(data_chunk: List[float]) -> dict:
    """วิเคราะห์ข้อมูล"""
    time.sleep(0.1)  # Simulate processing
    return {
        "count": len(data_chunk),
        "sum": sum(data_chunk),
        "mean": sum(data_chunk) / len(data_chunk),
        "max": max(data_chunk),
        "min": min(data_chunk),
        "std": math.sqrt(sum((x - sum(data_chunk)/len(data_chunk))**2 for x in data_chunk) / len(data_chunk))
    }

print("=== ProcessPoolExecutor ===")

# map()
large_numbers = [999983, 1000003, 1000033, 1000037, 1000039]

start = time.time()
with ProcessPoolExecutor(max_workers=4) as executor:
    results = list(executor.map(is_prime, large_numbers))  # is_prime จาก Part ก่อน

for n, is_p in zip(large_numbers, results):
    print(f"  {n}: {'prime' if is_p else 'not prime'}")

# submit() with as_completed
import random
data_chunks = [
    [random.gauss(50, 15) for _ in range(10000)]
    for _ in range(8)
]

start = time.time()
print("\nParallel data analysis:")
with ProcessPoolExecutor(max_workers=mp.cpu_count()) as executor:
    futures = {
        executor.submit(analyze_data, chunk): i
        for i, chunk in enumerate(data_chunks)
    }
    
    all_results = []
    for future in as_completed(futures):
        chunk_id = futures[future]
        stats = future.result()
        all_results.append(stats)
        print(f"  Chunk {chunk_id}: mean={stats['mean']:.2f}, std={stats['std']:.2f}")

# Aggregate results
total_count = sum(r["count"] for r in all_results)
overall_mean = sum(r["sum"] for r in all_results) / total_count
print(f"\nOverall: {total_count:,} items, mean={overall_mean:.2f}")
print(f"Time: {time.time()-start:.2f}s")

# Error handling
def risky_task(n: int) -> int:
    if n == 3:
        raise ValueError(f"Task {n} failed!")
    return n * 10

print("\n--- Error Handling ---")
with ProcessPoolExecutor(max_workers=4) as executor:
    futures = {executor.submit(risky_task, i): i for i in range(1, 7)}
    
    for future in as_completed(futures):
        task_id = futures[future]
        try:
            result = future.result()
            print(f"  Task {task_id}: {result}")
        except Exception as e:
            print(f"  Task {task_id} failed: {e}")
```

---

## 7. Process Pool Pattern

```python
import multiprocessing as mp
from multiprocessing import Pool
import time
import os
from typing import List, Dict

# Pattern: Worker Pool กับ initializer
worker_data = None  # Global ใน subprocess

def init_worker(data: dict) -> None:
    """เรียกครั้งเดียวต่อ process ตอน start"""
    global worker_data
    worker_data = data
    print(f"  Worker PID={os.getpid()} initialized with {len(data)} configs")

def process_item(item_id: int) -> dict:
    """ใช้ worker_data ที่ initialize แล้ว"""
    config = worker_data.get("config", {})
    multiplier = config.get("multiplier", 1)
    return {
        "id": item_id,
        "result": item_id * multiplier,
        "worker_pid": os.getpid()
    }

# Pool กับ initializer
shared_config = {"config": {"multiplier": 7, "mode": "fast"}}

with Pool(
    processes=3,
    initializer=init_worker,
    initargs=(shared_config,)
) as pool:
    items = list(range(1, 10))
    results = pool.map(process_item, items)

for r in results:
    print(f"  Item {r['id']}: {r['result']} (worker PID={r['worker_pid']})")

# Chunk size optimization
# Pool.map(func, iterable, chunksize=n)
# chunksize ใหญ่ = fewer IPC overhead แต่ load imbalance ถ้า tasks ไม่เท่ากัน

def variable_time_task(n: int) -> int:
    time.sleep(0.01 * (n % 5 + 1))  # เวลาไม่เท่ากัน
    return n ** 2

items = list(range(1, 21))

start = time.time()
with Pool(4) as pool:
    results = pool.map(variable_time_task, items, chunksize=1)
print(f"chunksize=1: {time.time()-start:.2f}s")

start = time.time()
with Pool(4) as pool:
    results = pool.map(variable_time_task, items, chunksize=5)
print(f"chunksize=5: {time.time()-start:.2f}s")

# callback
results_callback = []

def on_result(result: int) -> None:
    results_callback.append(result)
    print(f"  Callback: got {result}")

def on_error(error: Exception) -> None:
    print(f"  Error callback: {error}")

with Pool(processes=2) as pool:
    for n in [1, 2, 3, 4, 5]:
        pool.apply_async(
            lambda x: x**2,
            args=(n,),
            callback=on_result,
            error_callback=on_error
        )
    pool.close()
    pool.join()

print(f"Collected {len(results_callback)} results via callback")
```

---

## 8. เมื่อไหร่ควรใช้ Multiprocessing

```python
import time
import multiprocessing as mp
from concurrent.futures import ProcessPoolExecutor, ThreadPoolExecutor

# CPU-bound benchmark
def cpu_bound(n: int) -> float:
    """Pure CPU computation"""
    return sum(math.sqrt(i) for i in range(n))

# I/O-bound simulation
def io_bound(duration: float) -> str:
    time.sleep(duration)
    return f"done after {duration}s"

n_items = 8

print("=== CPU-bound Task Benchmark ===")
# Sequential
start = time.time()
[cpu_bound(500_000) for _ in range(n_items)]
print(f"Sequential:     {time.time()-start:.2f}s")

# ThreadPoolExecutor (GIL limits benefit)
start = time.time()
with ThreadPoolExecutor(max_workers=4) as ex:
    list(ex.map(cpu_bound, [500_000]*n_items))
print(f"ThreadPool:     {time.time()-start:.2f}s  (GIL: limited speedup)")

# ProcessPoolExecutor (True parallelism)
start = time.time()
with ProcessPoolExecutor(max_workers=4) as ex:
    list(ex.map(cpu_bound, [500_000]*n_items))
print(f"ProcessPool:    {time.time()-start:.2f}s  (FASTER!)")

print("\n=== I/O-bound Task Benchmark ===")
durations = [0.3] * n_items

# Sequential
start = time.time()
[io_bound(d) for d in durations]
print(f"Sequential:     {time.time()-start:.2f}s")

# ThreadPoolExecutor (good for I/O)
start = time.time()
with ThreadPoolExecutor(max_workers=4) as ex:
    list(ex.map(io_bound, durations))
print(f"ThreadPool:     {time.time()-start:.2f}s  (FASTER for I/O!)")

# ProcessPoolExecutor (overhead แพง, ไม่คุ้ม)
start = time.time()
with ProcessPoolExecutor(max_workers=4) as ex:
    list(ex.map(io_bound, durations))
print(f"ProcessPool:    {time.time()-start:.2f}s  (similar, but more overhead)")

# สรุปการเลือก
print("""
=== เมื่อไหร่ควรใช้อะไร? ===

I/O-bound tasks:
  ✅ threading + ThreadPoolExecutor
  ✅ asyncio (Part 034)
  ❌ multiprocessing (overhead แพงเกินไป)

CPU-bound tasks:
  ✅ multiprocessing + ProcessPoolExecutor
  ❌ threading (GIL จำกัด)
  ❌ asyncio (ไม่ได้ช่วย CPU)

Mixed workload:
  ✅ ProcessPoolExecutor สำหรับ CPU parts
  ✅ ThreadPoolExecutor สำหรับ I/O parts
  ✅ asyncio สำหรับ high concurrency I/O
""")
```

---

## 9. ตัวอย่างจริง: Parallel Image Processing

```python
import multiprocessing as mp
from concurrent.futures import ProcessPoolExecutor, as_completed
import time
import os
import math
from typing import List, Tuple, Dict
from dataclasses import dataclass

@dataclass
class ImageStats:
    filename: str
    width: int
    height: int
    total_pixels: int
    avg_brightness: float
    processing_time: float
    worker_pid: int

def simulate_image_load(filename: str) -> Tuple[int, int, List[int]]:
    """Simulate loading image pixels"""
    import random
    width = random.randint(800, 3840)
    height = random.randint(600, 2160)
    # Simulate pixel data (grayscale)
    pixels = [random.randint(0, 255) for _ in range(width * height // 100)]  # Sample
    return width, height, pixels

def process_image(filename: str) -> ImageStats:
    """Process a single image (CPU-bound)"""
    start = time.time()
    
    # Load image
    width, height, pixels = simulate_image_load(filename)
    
    # CPU-bound calculations
    total_pixels = width * height
    
    # Simulate complex operations
    brightness_sum = 0
    for p in pixels:
        brightness_sum += p
        # Apply gamma correction (simulate)
        _ = math.pow(p / 255.0, 2.2) * 255
    
    avg_brightness = brightness_sum / len(pixels)
    
    # Simulate more processing
    time.sleep(0.1)
    
    return ImageStats(
        filename=filename,
        width=width,
        height=height,
        total_pixels=total_pixels,
        avg_brightness=avg_brightness,
        processing_time=time.time()-start,
        worker_pid=os.getpid()
    )

class ImageProcessor:
    def __init__(self, max_workers: int = None) -> None:
        self.max_workers = max_workers or mp.cpu_count()
    
    def process_all(self, filenames: List[str]) -> List[ImageStats]:
        print(f"Processing {len(filenames)} images with {self.max_workers} workers...")
        
        results = []
        with ProcessPoolExecutor(max_workers=self.max_workers) as executor:
            futures = {executor.submit(process_image, f): f for f in filenames}
            
            for future in as_completed(futures):
                filename = futures[future]
                try:
                    stats = future.result()
                    results.append(stats)
                    print(f"  ✅ {stats.filename}: {stats.width}x{stats.height}, "
                          f"brightness={stats.avg_brightness:.1f}, "
                          f"time={stats.processing_time:.2f}s, "
                          f"PID={stats.worker_pid}")
                except Exception as e:
                    print(f"  ❌ {filename}: {e}")
        
        return results
    
    def generate_report(self, results: List[ImageStats]) -> None:
        if not results:
            return
        
        print(f"\n{'='*60}")
        print("IMAGE PROCESSING REPORT")
        print(f"{'='*60}")
        print(f"Images processed:     {len(results)}")
        print(f"Total pixels:         {sum(r.total_pixels for r in results):,}")
        
        total_time = sum(r.processing_time for r in results)
        avg_time = total_time / len(results)
        print(f"Avg processing time:  {avg_time:.3f}s per image")
        
        workers_used = len(set(r.worker_pid for r in results))
        print(f"Worker processes:     {workers_used}")
        
        print(f"\nLargest image: {max(results, key=lambda r: r.total_pixels).filename}")
        print(f"Brightest:     {max(results, key=lambda r: r.avg_brightness).filename}")
        print(f"Darkest:       {min(results, key=lambda r: r.avg_brightness).filename}")

# ใช้งาน
if __name__ == "__main__":
    filenames = [f"photo_{i:04d}.jpg" for i in range(1, 13)]
    
    processor = ImageProcessor(max_workers=4)
    
    start = time.time()
    results = processor.process_all(filenames)
    total_time = time.time() - start
    
    print(f"\nCompleted in {total_time:.2f}s")
    processor.generate_report(results)
```

---

## 10. สรุป Part 033

✅ **Multiprocessing** bypass GIL → True parallel CPU execution  
✅ **Process** class สร้าง subprocess ด้วย target function  
✅ **Pool** สำหรับ parallel map/apply_async  
✅ **Value, Array** share simple data ระหว่าง processes  
✅ **Manager** share complex objects (dict, list)  
✅ **Queue, Pipe** สำหรับ inter-process communication  
✅ **ProcessPoolExecutor** ง่ายกว่า Pool ใน concurrent.futures  
✅ **CPU-bound → multiprocessing** | **I/O-bound → threading/asyncio**  

---

## ➡️ ถัดไป: Part 034 - Async IO

*Part 033/100+ | Python Course - Beginner to World-Class*
