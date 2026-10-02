# Part 034: Async/Await - Asyncio
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Asynchronous Programming
- ใช้ async/await syntax
- asyncio event loop
- Coroutines, Tasks, Futures
- aiohttp, aiofiles
- ประยุกต์ใช้กับ FastAPI และ Django

---

## 1. Synchronous vs Asynchronous

```python
import time

# ❌ Synchronous - รอทีละอย่าง
def sync_task(name, delay):
    print(f"{name}: เริ่ม")
    time.sleep(delay)   # บล็อกทั้ง program!
    print(f"{name}: เสร็จ")

start = time.time()
sync_task("Task A", 2)
sync_task("Task B", 1)
sync_task("Task C", 3)
print(f"Total: {time.time()-start:.1f}s")  # ~6s

# ✅ Asynchronous - ทำพร้อมกัน
import asyncio

async def async_task(name, delay):
    print(f"{name}: เริ่ม")
    await asyncio.sleep(delay)   # ไม่บล็อก! ปล่อยให้ task อื่นทำ
    print(f"{name}: เสร็จ")
    return f"{name} result"

async def main():
    start = time.time()
    results = await asyncio.gather(
        async_task("Task A", 2),
        async_task("Task B", 1),
        async_task("Task C", 3),
    )
    print(f"Total: {time.time()-start:.1f}s")  # ~3s (เร็วกว่า!)
    print(f"Results: {results}")

asyncio.run(main())
```

---

## 2. Coroutines

```python
import asyncio

# Coroutine = function ที่ใช้ async def
async def greet(name):
    print(f"Hello, {name}!")
    await asyncio.sleep(1)
    print(f"Goodbye, {name}!")
    return f"greeted {name}"

# เรียก coroutine
async def main():
    # await - รอ coroutine ให้เสร็จ
    result = await greet("Alice")
    print(f"Result: {result}")
    
    # หลาย coroutines พร้อมกัน
    results = await asyncio.gather(
        greet("Bob"),
        greet("Charlie"),
        greet("Diana"),
    )
    print(f"All results: {results}")

asyncio.run(main())
```

---

## 3. Tasks

```python
import asyncio

async def download_file(url, delay):
    print(f"Downloading {url}...")
    await asyncio.sleep(delay)
    return f"Content of {url}"

async def main():
    # สร้าง Tasks (รันพร้อมกัน)
    task1 = asyncio.create_task(download_file("file1.txt", 2))
    task2 = asyncio.create_task(download_file("file2.txt", 1))
    task3 = asyncio.create_task(download_file("file3.txt", 3))
    
    # รอทุก task
    results = await asyncio.gather(task1, task2, task3)
    for r in results:
        print(r)
    
    # Task กับ timeout
    try:
        result = await asyncio.wait_for(
            download_file("slow.txt", 10),
            timeout=2.0
        )
    except asyncio.TimeoutError:
        print("Timeout!")

asyncio.run(main())
```

---

## 4. Async HTTP Requests ด้วย aiohttp

```python
import asyncio
import aiohttp  # pip install aiohttp

async def fetch_url(session, url):
    async with session.get(url) as response:
        return await response.json()

async def fetch_many_urls(urls):
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_url(session, url) for url in urls]
        results = await asyncio.gather(*tasks, return_exceptions=True)
    return results

# เปรียบเทียบ sync vs async
import requests

def sync_fetch(url):
    return requests.get(url).json()

async def main():
    urls = [
        "https://jsonplaceholder.typicode.com/posts/1",
        "https://jsonplaceholder.typicode.com/posts/2",
        "https://jsonplaceholder.typicode.com/posts/3",
    ]
    
    # Async (เร็วกว่า)
    import time
    start = time.time()
    results = await fetch_many_urls(urls)
    print(f"Async: {time.time()-start:.2f}s")
    
    # Sync (ช้ากว่า)
    start = time.time()
    sync_results = [sync_fetch(url) for url in urls]
    print(f"Sync: {time.time()-start:.2f}s")

asyncio.run(main())
```

---

## 5. Async File Operations

```python
import asyncio
import aiofiles  # pip install aiofiles

async def read_file_async(filename):
    async with aiofiles.open(filename, 'r') as f:
        return await f.read()

async def write_file_async(filename, content):
    async with aiofiles.open(filename, 'w') as f:
        await f.write(content)

async def process_files(filenames):
    tasks = [read_file_async(f) for f in filenames]
    contents = await asyncio.gather(*tasks)
    return contents

async def main():
    # เขียนไฟล์
    await write_file_async("test.txt", "Hello, Async World!")
    
    # อ่านไฟล์
    content = await read_file_async("test.txt")
    print(content)

asyncio.run(main())
```

---

## 6. Async กับ Database (SQLAlchemy Async)

```python
# สำหรับ FastAPI/async frameworks
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/db"

engine = create_async_engine(DATABASE_URL, echo=True)
AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_db():
    async with AsyncSessionLocal() as session:
        yield session

async def get_users(db: AsyncSession):
    from sqlalchemy import select
    result = await db.execute(select(User))
    return result.scalars().all()
```

---

## 7. Event Loop

```python
import asyncio

# ดู event loop
loop = asyncio.get_event_loop()
print(loop)

# สร้าง event loop เอง (ไม่แนะนำสำหรับ Python 3.10+)
# ใช้ asyncio.run() แทน

# asyncio.run() - วิธีที่แนะนำ
async def main():
    await asyncio.sleep(1)

asyncio.run(main())

# Semaphore - จำกัดจำนวน concurrent tasks
async def fetch_with_limit(semaphore, url):
    async with semaphore:  # จำกัดแค่ 5 พร้อมกัน
        await asyncio.sleep(1)  # simulate request
        return f"Result from {url}"

async def main():
    semaphore = asyncio.Semaphore(5)  # max 5 concurrent
    urls = [f"https://example.com/{i}" for i in range(20)]
    
    tasks = [fetch_with_limit(semaphore, url) for url in urls]
    results = await asyncio.gather(*tasks)
    print(f"Fetched {len(results)} URLs")

asyncio.run(main())
```

---

## 8. FastAPI ใช้ Async อย่างไร

```python
from fastapi import FastAPI
import asyncio

app = FastAPI()

@app.get("/sync")
def sync_endpoint():
    """Sync endpoint - FastAPI จะรันใน thread pool"""
    time.sleep(1)
    return {"message": "sync response"}

@app.get("/async")
async def async_endpoint():
    """Async endpoint - ไม่บล็อก event loop"""
    await asyncio.sleep(1)
    return {"message": "async response"}

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    # async database query
    user = await db.get_user(user_id)
    return user
```

---

## 9. สรุป Part 034

✅ **async/await** - syntax สำหรับ async programming  
✅ **asyncio.gather()** - รัน coroutines พร้อมกัน  
✅ **asyncio.create_task()** - สร้าง background tasks  
✅ **aiohttp** - async HTTP requests  
✅ **aiofiles** - async file I/O  
✅ **Semaphore** - จำกัด concurrency  
✅ **Event Loop** - core ของ asyncio  

### เมื่อใช้ Async:
- I/O-bound tasks (HTTP, database, file)
- FastAPI (async framework)
- ต้องการ high concurrency

### เมื่อ KHÔNG ใช้ Async:
- CPU-bound tasks (ใช้ multiprocessing แทน)
- โค้ดง่ายๆ ที่ไม่ต้องการ concurrency

---

## ➡️ ถัดไป: Part 035 - Database ด้วย SQLite

*Part 034/100+ | Python Course - Beginner to World-Class*
