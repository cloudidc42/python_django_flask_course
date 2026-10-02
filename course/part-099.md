# Part 099: High Performance Python

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ asyncio best practices สำหรับ high-performance apps
- ใช้ Connection Pooling อย่างมีประสิทธิภาพ
- Optimize Database Queries และ N+1 Problems
- ใช้ Profiling tools หาจุด bottleneck
- Memory Optimization techniques
- แนะนำ Cython และ C Extensions
- ใช้ Multiprocessing สำหรับ CPU-intensive tasks

---

## 1. Asyncio Best Practices

```python
# asyncio/best_practices.py
"""
asyncio Best Practices สำหรับ Python Web Applications
"""
import asyncio
import time
import httpx
import aiofiles
from typing import List, Any, Coroutine
import logging

logger = logging.getLogger(__name__)


# ❌ ไม่ดี: Sequential async calls (blocking อยู่ดี)
async def fetch_data_sequential(user_ids: List[int]) -> List[dict]:
    """ทำงาน sequential ทั้งที่ควรทำ parallel"""
    results = []
    async with httpx.AsyncClient() as client:
        for user_id in user_ids:
            response = await client.get(f"https://api.example.com/users/{user_id}")
            results.append(response.json())
    return results


# ✅ ดีกว่า: Concurrent async calls
async def fetch_data_concurrent(user_ids: List[int]) -> List[dict]:
    """ทำงาน concurrent - เร็วกว่ามาก"""
    async with httpx.AsyncClient() as client:
        tasks = [
            client.get(f"https://api.example.com/users/{user_id}")
            for user_id in user_ids
        ]
        responses = await asyncio.gather(*tasks, return_exceptions=True)
    
    results = []
    for i, response in enumerate(responses):
        if isinstance(response, Exception):
            logger.error(f"Failed to fetch user {user_ids[i]}: {response}")
            results.append(None)
        else:
            results.append(response.json())
    
    return results


# ✅ ดีที่สุด: Concurrent with timeout and semaphore
async def fetch_data_controlled(
    user_ids: List[int],
    max_concurrent: int = 10,
    timeout: float = 5.0
) -> List[dict]:
    """ควบคุม concurrency เพื่อไม่ overload server"""
    semaphore = asyncio.Semaphore(max_concurrent)
    
    async def fetch_one(client: httpx.AsyncClient, user_id: int) -> dict:
        async with semaphore:
            try:
                response = await asyncio.wait_for(
                    client.get(f"https://api.example.com/users/{user_id}"),
                    timeout=timeout
                )
                return response.json()
            except asyncio.TimeoutError:
                logger.warning(f"Timeout fetching user {user_id}")
                return None
            except Exception as e:
                logger.error(f"Error fetching user {user_id}: {e}")
                return None
    
    async with httpx.AsyncClient() as client:
        tasks = [fetch_one(client, uid) for uid in user_ids]
        return await asyncio.gather(*tasks)


# Task Groups (Python 3.11+)
async def fetch_with_task_group(user_ids: List[int]) -> List[dict]:
    """ใช้ TaskGroup (Python 3.11+) สำหรับ structured concurrency"""
    results = {}
    
    async with asyncio.TaskGroup() as tg:
        async def fetch_one(user_id: int):
            async with httpx.AsyncClient() as client:
                response = await client.get(f"https://api.example.com/users/{user_id}")
                results[user_id] = response.json()
        
        for user_id in user_ids:
            tg.create_task(fetch_one(user_id))
    
    return [results.get(uid) for uid in user_ids]


# Background tasks
class BackgroundTaskRunner:
    """จัดการ background tasks อย่างมีประสิทธิภาพ"""
    
    def __init__(self, max_workers: int = 10):
        self.semaphore = asyncio.Semaphore(max_workers)
        self.running_tasks: set = set()
    
    async def run(self, coro: Coroutine, *, ignore_errors: bool = True) -> asyncio.Task:
        """Run coroutine ใน background"""
        async def _wrapper():
            async with self.semaphore:
                try:
                    await coro
                except Exception as e:
                    if not ignore_errors:
                        raise
                    logger.error(f"Background task error: {e}")
        
        task = asyncio.create_task(_wrapper())
        self.running_tasks.add(task)
        task.add_done_callback(self.running_tasks.discard)
        return task
    
    async def wait_all(self):
        """รอให้ทุก tasks เสร็จ"""
        if self.running_tasks:
            await asyncio.gather(*self.running_tasks, return_exceptions=True)


# Async Context Manager สำหรับ timing
class AsyncTimer:
    def __init__(self, name: str = ""):
        self.name = name
        self.elapsed = 0
    
    async def __aenter__(self):
        self.start = time.perf_counter()
        return self
    
    async def __aexit__(self, *args):
        self.elapsed = time.perf_counter() - self.start
        logger.info(f"{self.name}: {self.elapsed:.3f}s")


# ตัวอย่าง async pipeline
async def process_pipeline(items: List[Any]) -> List[Any]:
    """
    Async pipeline สำหรับประมวลผลข้อมูลเป็นขั้นตอน
    แต่ละขั้นตอนทำงาน concurrent
    """
    
    async def stage1_fetch(item) -> dict:
        """ดึงข้อมูลเพิ่มเติม"""
        await asyncio.sleep(0.01)  # simulate I/O
        return {"id": item, "data": f"fetched_{item}"}
    
    async def stage2_transform(item: dict) -> dict:
        """แปลงข้อมูล"""
        await asyncio.sleep(0.005)  # simulate processing
        return {**item, "transformed": True}
    
    async def stage3_store(item: dict) -> bool:
        """บันทึกข้อมูล"""
        await asyncio.sleep(0.01)  # simulate I/O
        return True
    
    # Stage 1: Fetch (concurrent)
    async with AsyncTimer("Stage 1 - Fetch"):
        stage1_results = await asyncio.gather(*[stage1_fetch(item) for item in items])
    
    # Stage 2: Transform (concurrent)
    async with AsyncTimer("Stage 2 - Transform"):
        stage2_results = await asyncio.gather(*[stage2_transform(r) for r in stage1_results])
    
    # Stage 3: Store (concurrent with limit)
    semaphore = asyncio.Semaphore(5)  # max 5 concurrent DB writes
    async with AsyncTimer("Stage 3 - Store"):
        async def store_with_limit(item):
            async with semaphore:
                return await stage3_store(item)
        
        await asyncio.gather(*[store_with_limit(r) for r in stage2_results])
    
    return stage2_results
```

---

## 2. Connection Pooling

```python
# connection_pool/database_pool.py
"""
Connection Pooling สำหรับ Python Applications
"""
import asyncpg
import aiomysql
import redis.asyncio as aioredis
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker
from contextlib import asynccontextmanager
from typing import AsyncGenerator
import os


# PostgreSQL Connection Pool ด้วย asyncpg
class PostgreSQLPool:
    """High-performance PostgreSQL connection pool"""
    
    def __init__(
        self,
        dsn: str,
        min_size: int = 5,
        max_size: int = 20,
        max_inactive_connection_lifetime: float = 300
    ):
        self.dsn = dsn
        self.min_size = min_size
        self.max_size = max_size
        self.max_inactive_lifetime = max_inactive_connection_lifetime
        self._pool = None
    
    async def initialize(self):
        """สร้าง connection pool"""
        self._pool = await asyncpg.create_pool(
            dsn=self.dsn,
            min_size=self.min_size,
            max_size=self.max_size,
            max_inactive_connection_lifetime=self.max_inactive_lifetime,
            command_timeout=30,
            statement_cache_size=100,  # cache prepared statements
        )
        print(f"PostgreSQL pool initialized: {self.min_size}-{self.max_size} connections")
    
    async def close(self):
        """ปิด connection pool"""
        if self._pool:
            await self._pool.close()
    
    @asynccontextmanager
    async def acquire(self) -> AsyncGenerator[asyncpg.Connection, None]:
        """ยืม connection จาก pool"""
        async with self._pool.acquire() as conn:
            yield conn
    
    async def execute(self, query: str, *args) -> str:
        """Execute query"""
        async with self._pool.acquire() as conn:
            return await conn.execute(query, *args)
    
    async def fetch(self, query: str, *args) -> list:
        """Fetch multiple rows"""
        async with self._pool.acquire() as conn:
            return await conn.fetch(query, *args)
    
    async def fetchrow(self, query: str, *args) -> dict:
        """Fetch single row"""
        async with self._pool.acquire() as conn:
            return await conn.fetchrow(query, *args)
    
    async def fetchval(self, query: str, *args):
        """Fetch single value"""
        async with self._pool.acquire() as conn:
            return await conn.fetchval(query, *args)
    
    async def executemany(self, query: str, args_list: list):
        """Execute query กับหลาย arguments"""
        async with self._pool.acquire() as conn:
            await conn.executemany(query, args_list)
    
    @asynccontextmanager
    async def transaction(self) -> AsyncGenerator[asyncpg.Connection, None]:
        """Transaction context manager"""
        async with self._pool.acquire() as conn:
            async with conn.transaction():
                yield conn
    
    def get_stats(self) -> dict:
        """ดู pool statistics"""
        return {
            "size": self._pool.get_size(),
            "min_size": self._pool.get_min_size(),
            "max_size": self._pool.get_max_size(),
            "idle_connections": self._pool.get_idle_size(),
        }


# SQLAlchemy Async Pool
DATABASE_URL = os.getenv("DATABASE_URL", "postgresql+asyncpg://user:pass@localhost/db")

engine = create_async_engine(
    DATABASE_URL,
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True,          # ตรวจสอบ connection ก่อนใช้
    pool_recycle=3600,           # recycle connections ทุก 1 ชั่วโมง
    pool_timeout=30,             # timeout รอ connection
    echo=False,                  # ไม่ log SQL ใน production
)

AsyncSessionLocal = sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)


@asynccontextmanager
async def get_db_session() -> AsyncGenerator[AsyncSession, None]:
    """Context manager สำหรับ database session"""
    session = AsyncSessionLocal()
    try:
        yield session
        await session.commit()
    except Exception:
        await session.rollback()
        raise
    finally:
        await session.close()


# Redis Connection Pool
class RedisPool:
    """Redis connection pool"""
    
    def __init__(self, url: str = "redis://localhost:6379"):
        self.url = url
        self._pool = None
    
    async def initialize(self):
        self._pool = aioredis.ConnectionPool.from_url(
            self.url,
            max_connections=50,
            decode_responses=True
        )
    
    def get_client(self) -> aioredis.Redis:
        return aioredis.Redis(connection_pool=self._pool)
    
    async def close(self):
        await self._pool.disconnect()


# HTTP Connection Pool
class HTTPConnectionPool:
    """HTTP client ที่มี connection pooling"""
    
    def __init__(self):
        self._client: httpx.AsyncClient = None
    
    async def initialize(self):
        """สร้าง HTTP client พร้อม connection pool"""
        import httpx
        
        transport = httpx.AsyncHTTPTransport(
            limits=httpx.Limits(
                max_connections=100,
                max_keepalive_connections=20,
                keepalive_expiry=30
            )
        )
        
        self._client = httpx.AsyncClient(
            transport=transport,
            timeout=httpx.Timeout(connect=5.0, read=30.0, write=10.0, pool=5.0)
        )
    
    async def get(self, url: str, **kwargs):
        return await self._client.get(url, **kwargs)
    
    async def post(self, url: str, **kwargs):
        return await self._client.post(url, **kwargs)
    
    async def close(self):
        await self._client.aclose()
```

---

## 3. Query Optimization

```python
# optimization/query_optimizer.py
"""
Database Query Optimization Techniques
"""
from sqlalchemy import select, func, text, Index
from sqlalchemy.orm import selectinload, joinedload, subqueryload, contains_eager
from sqlalchemy.ext.asyncio import AsyncSession
from typing import List, Optional


# Models
from sqlalchemy import Column, Integer, String, ForeignKey, Float, DateTime
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import relationship
from datetime import datetime

Base = declarative_base()


class User(Base):
    __tablename__ = "users"
    __table_args__ = (
        Index("idx_users_email", "email"),
        Index("idx_users_created_at", "created_at"),
    )
    
    id = Column(Integer, primary_key=True)
    email = Column(String, nullable=False)
    name = Column(String)
    created_at = Column(DateTime, default=datetime.utcnow)
    orders = relationship("Order", back_populates="user", lazy="select")


class Product(Base):
    __tablename__ = "products"
    __table_args__ = (
        Index("idx_products_category", "category"),
        Index("idx_products_price", "price"),
        Index("idx_products_category_price", "category", "price"),  # Composite index
    )
    
    id = Column(Integer, primary_key=True)
    name = Column(String)
    price = Column(Float)
    category = Column(String)
    order_items = relationship("OrderItem", back_populates="product")


class Order(Base):
    __tablename__ = "orders"
    __table_args__ = (
        Index("idx_orders_user_id", "user_id"),
        Index("idx_orders_status", "status"),
        Index("idx_orders_created_at", "created_at"),
    )
    
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    status = Column(String, default="pending")
    total = Column(Float)
    created_at = Column(DateTime, default=datetime.utcnow)
    user = relationship("User", back_populates="orders")
    items = relationship("OrderItem", back_populates="order")


class OrderItem(Base):
    __tablename__ = "order_items"
    
    id = Column(Integer, primary_key=True)
    order_id = Column(Integer, ForeignKey("orders.id"))
    product_id = Column(Integer, ForeignKey("products.id"))
    quantity = Column(Integer)
    price = Column(Float)
    order = relationship("Order", back_populates="items")
    product = relationship("Product", back_populates="order_items")


class QueryOptimizer:
    """ตัวอย่าง query optimization techniques"""
    
    def __init__(self, session: AsyncSession):
        self.db = session
    
    # ❌ N+1 Problem
    async def get_orders_n_plus_1(self, user_id: int) -> List[Order]:
        """ปัญหา N+1: แต่ละ order จะ query ไปดึง items แยก"""
        result = await self.db.execute(
            select(Order).where(Order.user_id == user_id)
        )
        orders = result.scalars().all()
        
        # แต่ละ order access .items จะ trigger query แยก!
        # ถ้ามี 100 orders = 101 queries
        for order in orders:
            _ = order.items  # N+1 Problem
        
        return orders
    
    # ✅ แก้ด้วย Eager Loading
    async def get_orders_with_eager_loading(self, user_id: int) -> List[Order]:
        """แก้ N+1 ด้วย joinedload"""
        result = await self.db.execute(
            select(Order)
            .where(Order.user_id == user_id)
            .options(
                joinedload(Order.items).joinedload(OrderItem.product)
            )
        )
        return result.unique().scalars().all()
    
    # ✅ selectinload สำหรับ one-to-many (ดีกว่า joinedload สำหรับ collections)
    async def get_orders_selectin(self, user_id: int) -> List[Order]:
        """ใช้ selectinload - 2 queries แทน N+1"""
        result = await self.db.execute(
            select(Order)
            .where(Order.user_id == user_id)
            .options(
                selectinload(Order.items).selectinload(OrderItem.product)
            )
        )
        return result.scalars().all()
    
    # ✅ Pagination ที่ถูกต้อง
    async def get_orders_paginated(
        self,
        user_id: int,
        page: int = 1,
        page_size: int = 20
    ) -> dict:
        """Pagination ที่มีประสิทธิภาพ"""
        # Count query
        count_result = await self.db.execute(
            select(func.count(Order.id)).where(Order.user_id == user_id)
        )
        total = count_result.scalar()
        
        # Data query
        result = await self.db.execute(
            select(Order)
            .where(Order.user_id == user_id)
            .order_by(Order.created_at.desc())
            .offset((page - 1) * page_size)
            .limit(page_size)
            .options(selectinload(Order.items))
        )
        orders = result.scalars().all()
        
        return {
            "data": orders,
            "total": total,
            "page": page,
            "page_size": page_size,
            "total_pages": (total + page_size - 1) // page_size
        }
    
    # ✅ Keyset Pagination (สำหรับ large datasets)
    async def get_orders_keyset_paginated(
        self,
        last_id: Optional[int] = None,
        page_size: int = 20
    ) -> List[Order]:
        """Keyset pagination - เร็วกว่า OFFSET สำหรับ large datasets"""
        query = select(Order).order_by(Order.id.asc()).limit(page_size)
        
        if last_id:
            query = query.where(Order.id > last_id)
        
        result = await self.db.execute(query)
        return result.scalars().all()
    
    # ✅ Bulk operations
    async def bulk_update_order_status(self, order_ids: List[int], new_status: str):
        """อัพเดทหลาย orders พร้อมกัน"""
        from sqlalchemy import update
        
        await self.db.execute(
            update(Order)
            .where(Order.id.in_(order_ids))
            .values(status=new_status)
        )
    
    # ✅ Aggregate queries
    async def get_sales_summary(self) -> dict:
        """ดึงสถิติยอดขาย"""
        result = await self.db.execute(
            select(
                Order.status,
                func.count(Order.id).label("count"),
                func.sum(Order.total).label("total_amount"),
                func.avg(Order.total).label("avg_amount")
            )
            .group_by(Order.status)
        )
        
        return {
            row.status: {
                "count": row.count,
                "total": float(row.total_amount or 0),
                "average": float(row.avg_amount or 0)
            }
            for row in result
        }
    
    # ✅ Raw SQL สำหรับ complex queries
    async def get_top_customers(self, limit: int = 10) -> List[dict]:
        """ดึง top customers ด้วย raw SQL"""
        result = await self.db.execute(
            text("""
                SELECT 
                    u.id,
                    u.name,
                    u.email,
                    COUNT(o.id) as order_count,
                    SUM(o.total) as total_spent
                FROM users u
                LEFT JOIN orders o ON o.user_id = u.id 
                    AND o.status = 'completed'
                GROUP BY u.id, u.name, u.email
                HAVING COUNT(o.id) > 0
                ORDER BY total_spent DESC
                LIMIT :limit
            """),
            {"limit": limit}
        )
        
        return [dict(row._mapping) for row in result]
```

---

## 4. Profiling Tools

```python
# profiling/profiler.py
"""
Python Profiling Tools:
1. cProfile - built-in profiler
2. line_profiler - line-by-line profiling
3. memory_profiler - memory usage
4. py-spy - low-overhead profiler
5. Pyflame - flame graph profiler
"""
import cProfile
import pstats
import io
import time
import tracemalloc
import functools
from typing import Callable
import asyncio


# cProfile Profiler
def profile_function(func: Callable) -> Callable:
    """Decorator สำหรับ profile function ด้วย cProfile"""
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        profiler = cProfile.Profile()
        profiler.enable()
        
        result = func(*args, **kwargs)
        
        profiler.disable()
        
        # Print stats
        stream = io.StringIO()
        stats = pstats.Stats(profiler, stream=stream)
        stats.sort_stats("cumulative")
        stats.print_stats(20)  # Top 20 functions
        
        print(f"\n=== Profile for {func.__name__} ===")
        print(stream.getvalue())
        
        return result
    return wrapper


class PerformanceProfiler:
    """Profiler สำหรับ web application"""
    
    def __init__(self, threshold_ms: float = 100):
        self.threshold_ms = threshold_ms
        self.slow_requests = []
    
    def profile_request(self, func: Callable) -> Callable:
        """Profile HTTP requests"""
        @functools.wraps(func)
        async def async_wrapper(*args, **kwargs):
            start = time.perf_counter()
            
            # Memory tracking
            tracemalloc.start()
            
            try:
                result = await func(*args, **kwargs)
                elapsed_ms = (time.perf_counter() - start) * 1000
                
                current, peak = tracemalloc.get_traced_memory()
                tracemalloc.stop()
                
                if elapsed_ms > self.threshold_ms:
                    request = args[0] if args else None
                    path = getattr(getattr(request, 'url', None), 'path', 'unknown')
                    
                    slow_info = {
                        "path": path,
                        "elapsed_ms": round(elapsed_ms, 2),
                        "peak_memory_mb": round(peak / 1024 / 1024, 2),
                    }
                    self.slow_requests.append(slow_info)
                    print(f"⚠️  Slow request: {slow_info}")
                
                return result
            finally:
                if tracemalloc.is_tracing():
                    tracemalloc.stop()
        
        return async_wrapper


# Memory Profiler
class MemoryProfiler:
    """วิเคราะห์การใช้ memory"""
    
    def __init__(self):
        self.snapshots = []
    
    def take_snapshot(self, label: str = ""):
        """บันทึก memory snapshot"""
        tracemalloc.start()
        snapshot = tracemalloc.take_snapshot()
        self.snapshots.append((label, snapshot))
        return snapshot
    
    def compare_snapshots(self, snapshot1_label: str = None, snapshot2_label: str = None):
        """เปรียบเทียบ 2 snapshots"""
        if len(self.snapshots) < 2:
            print("Need at least 2 snapshots")
            return
        
        s1 = self.snapshots[-2][1]
        s2 = self.snapshots[-1][1]
        
        stats = s2.compare_to(s1, "lineno")
        
        print("\n=== Memory Comparison ===")
        for stat in stats[:10]:
            print(f"  {stat}")
    
    def get_top_memory_consumers(self, limit: int = 10):
        """ดู top memory consumers"""
        tracemalloc.start()
        snapshot = tracemalloc.take_snapshot()
        
        stats = snapshot.statistics("lineno")
        
        print("\n=== Top Memory Consumers ===")
        for stat in stats[:limit]:
            print(f"  {stat.size / 1024:.1f} KB: {stat}")
    
    @staticmethod
    def get_current_memory_mb() -> float:
        """ดู memory usage ปัจจุบัน"""
        import psutil
        import os
        process = psutil.Process(os.getpid())
        return process.memory_info().rss / 1024 / 1024


# ตัวอย่างการใช้ profiling
@profile_function
def expensive_computation():
    """ฟังก์ชันที่ต้อง profile"""
    result = 0
    for i in range(1_000_000):
        result += i * i
    return result


# Line-by-line timing
class LineTimer:
    """วัดเวลาแต่ละส่วนของโค้ด"""
    
    def __init__(self, name: str = ""):
        self.name = name
        self.checkpoints = []
        self.start_time = None
    
    def start(self):
        self.start_time = time.perf_counter()
        self.checkpoints = []
        return self
    
    def checkpoint(self, label: str):
        if self.start_time:
            elapsed = (time.perf_counter() - self.start_time) * 1000
            self.checkpoints.append((label, elapsed))
    
    def report(self):
        print(f"\n=== Timer: {self.name} ===")
        prev = 0
        for label, elapsed in self.checkpoints:
            delta = elapsed - prev
            print(f"  {label}: {elapsed:.2f}ms (+{delta:.2f}ms)")
            prev = elapsed
```

---

## 5. Memory Optimization

```python
# optimization/memory_opt.py
"""
Memory Optimization Techniques:
1. __slots__ สำหรับ reduce class overhead
2. Generator expressions แทน list comprehensions
3. itertools สำหรับ lazy evaluation
4. weakref สำหรับ cache ที่ไม่ block GC
5. numpy arrays แทน Python lists
"""
import sys
import gc
import weakref
from typing import Iterator, Generator, List
import itertools


# __slots__ ลด memory overhead ของ class instances
class UserWithSlots:
    """Class ที่ใช้ __slots__ - ประหยัด memory"""
    __slots__ = ["id", "email", "name", "age"]
    
    def __init__(self, id: int, email: str, name: str, age: int):
        self.id = id
        self.email = email
        self.name = name
        self.age = age


class UserWithoutSlots:
    """Class ปกติ - ใช้ __dict__"""
    def __init__(self, id: int, email: str, name: str, age: int):
        self.id = id
        self.email = email
        self.name = name
        self.age = age


def compare_memory_usage():
    """เปรียบเทียบ memory usage"""
    user_slots = UserWithSlots(1, "test@example.com", "Test User", 25)
    user_no_slots = UserWithoutSlots(1, "test@example.com", "Test User", 25)
    
    print(f"Size with __slots__: {sys.getsizeof(user_slots)} bytes")
    print(f"Size without __slots__: {sys.getsizeof(user_no_slots)} bytes")
    # __slots__ ประหยัดประมาณ 50-200 bytes ต่อ instance


# Generator สำหรับ lazy evaluation
def process_large_dataset(filename: str) -> Generator[dict, None, None]:
    """Process large file โดยไม่โหลดทั้งหมดเข้า memory"""
    import json
    
    with open(filename, "r") as f:
        for line in f:  # อ่านทีละบรรทัด (lazy)
            try:
                yield json.loads(line.strip())
            except json.JSONDecodeError:
                continue


def calculate_stats(data_generator: Generator) -> dict:
    """คำนวณ statistics โดยไม่เก็บข้อมูลทั้งหมดใน memory"""
    total = 0
    count = 0
    min_val = float("inf")
    max_val = float("-inf")
    
    for item in data_generator:
        value = item.get("value", 0)
        total += value
        count += 1
        min_val = min(min_val, value)
        max_val = max(max_val, value)
    
    return {
        "total": total,
        "count": count,
        "average": total / count if count else 0,
        "min": min_val,
        "max": max_val
    }


# WeakRef Cache - ไม่ block garbage collection
class WeakRefCache:
    """Cache ที่ใช้ weakref - ป้องกัน memory leak"""
    
    def __init__(self):
        self._cache = weakref.WeakValueDictionary()
    
    def get(self, key: str):
        return self._cache.get(key)
    
    def set(self, key: str, value):
        self._cache[key] = value
    
    def __len__(self):
        return len(self._cache)


# Memory-efficient data structures
class MemoryEfficientQueue:
    """Queue ที่ประหยัด memory ด้วย deque"""
    
    def __init__(self, maxsize: int = 1000):
        from collections import deque
        self._queue = deque(maxlen=maxsize)  # auto-drop oldest items
    
    def enqueue(self, item):
        self._queue.append(item)
    
    def dequeue(self):
        if self._queue:
            return self._queue.popleft()
        return None
    
    def __len__(self):
        return len(self._queue)


# ใช้ array module แทน list สำหรับ homogeneous data
import array

def use_array_instead_of_list():
    """array.array ประหยัด memory กว่า list สำหรับตัวเลข"""
    python_list = list(range(1_000_000))
    int_array = array.array("l", range(1_000_000))  # signed long
    
    print(f"List size: {sys.getsizeof(python_list) / 1024:.1f} KB")
    print(f"Array size: {sys.getsizeof(int_array) / 1024:.1f} KB")
```

---

## 6. Cython และ C Extensions

```python
# cython_example/fast_math.pyx - Cython source file
"""
Cython: Python ที่ compile เป็น C สำหรับ performance

การติดตั้ง:
pip install cython

สร้าง setup.py แล้วรัน:
python setup.py build_ext --inplace
"""

# fast_math.pyx
"""
# cython: language_level=3
# cython: boundscheck=False  # ปิด bounds checking
# cython: wraparound=False   # ปิด negative index
# cython: nonecheck=False    # ปิด None checking

import cython
from libc.math cimport sqrt, pow


def calculate_distance(double x1, double y1, double x2, double y2) -> double:
    \"\"\"คำนวณ Euclidean distance (เร็วกว่า Python ปกติ ~50x)\"\"\"
    cdef double dx = x2 - x1
    cdef double dy = y2 - y1
    return sqrt(dx * dx + dy * dy)


def sum_squares(int n) -> long:
    \"\"\"คำนวณผลรวมกำลังสอง (เร็วกว่า Python ~100x)\"\"\"
    cdef long result = 0
    cdef int i
    for i in range(n):
        result += i * i
    return result


def process_array(double[:] data) -> double:
    \"\"\"ประมวลผล NumPy array (typed memory view)\"\"\"
    cdef int n = len(data)
    cdef double total = 0.0
    cdef int i
    
    for i in range(n):
        total += data[i] * data[i]
    
    return sqrt(total)
"""

# setup.py สำหรับ compile Cython
"""
from setuptools import setup
from Cython.Build import cythonize
import numpy

setup(
    ext_modules=cythonize(
        "fast_math.pyx",
        compiler_directives={
            "language_level": "3",
            "boundscheck": False,
            "wraparound": False,
        }
    ),
    include_dirs=[numpy.get_include()]
)
"""


# Pure Python เพื่อเปรียบเทียบ
def py_calculate_distance(x1, y1, x2, y2):
    import math
    return math.sqrt((x2-x1)**2 + (y2-y1)**2)


def py_sum_squares(n):
    return sum(i*i for i in range(n))


# ctypes: ใช้ C library โดยตรง
import ctypes
import os

def use_c_library():
    """ใช้ C function ผ่าน ctypes"""
    # โหลด C standard library
    libc = ctypes.CDLL("libc.so.6")
    
    # ตั้งค่า return type
    libc.sqrt.restype = ctypes.c_double
    libc.sqrt.argtypes = [ctypes.c_double]
    
    result = libc.sqrt(2.0)
    print(f"sqrt(2) = {result}")


# NumPy สำหรับ vectorized operations
import numpy as np

def numpy_vs_python():
    """เปรียบเทียบ NumPy กับ Python loop"""
    import time
    
    n = 1_000_000
    data = list(range(n))
    
    # Python loop
    start = time.perf_counter()
    result = [x * x for x in data]
    py_time = time.perf_counter() - start
    
    # NumPy vectorized
    np_data = np.array(data)
    start = time.perf_counter()
    result = np_data ** 2
    np_time = time.perf_counter() - start
    
    print(f"Python: {py_time*1000:.1f}ms")
    print(f"NumPy: {np_time*1000:.1f}ms")
    print(f"Speedup: {py_time/np_time:.1f}x")
```

---

## 7. Multiprocessing สำหรับ CPU Tasks

```python
# multiprocessing_cpu/cpu_tasks.py
"""
Multiprocessing สำหรับ CPU-intensive tasks
(asyncio ไม่ช่วย CPU-bound tasks เพราะ GIL)
"""
import multiprocessing as mp
from multiprocessing import Pool, Queue, Process, Manager
import concurrent.futures
import os
import time
import numpy as np
from typing import List, Callable, Any


def cpu_intensive_task(data: list) -> float:
    """CPU-intensive task: คำนวณ statistics"""
    arr = np.array(data)
    result = 0
    for _ in range(100):  # จำลอง heavy computation
        result += np.sum(arr ** 2) / len(arr)
    return result


def parallel_processing_pool(data_chunks: List[list]) -> List[float]:
    """ประมวลผล data chunks แบบ parallel ด้วย Pool"""
    num_workers = os.cpu_count()
    
    with Pool(processes=num_workers) as pool:
        results = pool.map(cpu_intensive_task, data_chunks)
    
    return results


class ProcessPoolManager:
    """จัดการ process pool สำหรับ web application"""
    
    def __init__(self, max_workers: int = None):
        self.max_workers = max_workers or os.cpu_count()
        self.executor = concurrent.futures.ProcessPoolExecutor(
            max_workers=self.max_workers,
            initializer=worker_initializer,
            initargs=(shared_config,)
        )
    
    def submit(self, func: Callable, *args, **kwargs):
        """Submit task ไปยัง process pool"""
        return self.executor.submit(func, *args, **kwargs)
    
    def map(self, func: Callable, iterable, timeout: float = None):
        """Map function ไปยัง iterable"""
        return list(self.executor.map(func, iterable, timeout=timeout))
    
    async def submit_async(self, loop, func: Callable, *args):
        """Submit task แบบ async"""
        return await loop.run_in_executor(self.executor, func, *args)
    
    def shutdown(self):
        self.executor.shutdown(wait=True)


shared_config = {"model_path": "/models/"}


def worker_initializer(config: dict):
    """Initialize worker process (เช่น โหลด ML model)"""
    global worker_config
    worker_config = config
    print(f"Worker {os.getpid()} initialized")


# Chunking สำหรับ large datasets
def chunk_data(data: List, chunk_size: int) -> List[List]:
    """แบ่ง data เป็น chunks"""
    return [data[i:i+chunk_size] for i in range(0, len(data), chunk_size)]


async def process_large_dataset_parallel(data: List, chunk_size: int = 1000):
    """Process large dataset ด้วย multiprocessing + asyncio"""
    import asyncio
    
    chunks = chunk_data(data, chunk_size)
    loop = asyncio.get_event_loop()
    
    # ใช้ ProcessPoolExecutor กับ asyncio
    with concurrent.futures.ProcessPoolExecutor(max_workers=os.cpu_count()) as executor:
        tasks = [
            loop.run_in_executor(executor, cpu_intensive_task, chunk)
            for chunk in chunks
        ]
        results = await asyncio.gather(*tasks)
    
    return results


# Worker Queue Pattern
class WorkerQueueSystem:
    """ระบบ worker queue สำหรับ background processing"""
    
    def __init__(self, num_workers: int = 4):
        self.num_workers = num_workers
        self.task_queue = mp.Queue()
        self.result_queue = mp.Queue()
        self.workers: List[Process] = []
        self._stop_event = mp.Event()
    
    def start(self):
        """เริ่ม workers"""
        for i in range(self.num_workers):
            worker = Process(
                target=self._worker_loop,
                args=(i, self.task_queue, self.result_queue, self._stop_event),
                daemon=True
            )
            worker.start()
            self.workers.append(worker)
        print(f"Started {self.num_workers} workers")
    
    def stop(self):
        """หยุด workers"""
        self._stop_event.set()
        for worker in self.workers:
            worker.join(timeout=5)
    
    @staticmethod
    def _worker_loop(
        worker_id: int,
        task_queue: Queue,
        result_queue: Queue,
        stop_event: mp.Event
    ):
        """Worker loop"""
        print(f"Worker {worker_id} (PID: {os.getpid()}) started")
        
        while not stop_event.is_set():
            try:
                task = task_queue.get(timeout=1)
                task_id, func_name, args = task
                
                # ประมวลผล task
                result = cpu_intensive_task(args)
                result_queue.put((task_id, result, None))
                
            except Exception as e:
                if "Empty" not in type(e).__name__:
                    result_queue.put((None, None, str(e)))
    
    def submit_task(self, task_id: str, args: Any) -> str:
        """Submit task ไปยัง queue"""
        self.task_queue.put((task_id, "cpu_task", args))
        return task_id
    
    def get_result(self, timeout: float = 30) -> tuple:
        """รับผลลัพธ์จาก queue"""
        return self.result_queue.get(timeout=timeout)


# ตัวอย่างการใช้งาน
def demo_multiprocessing():
    """Demo การใช้ multiprocessing"""
    print(f"CPU cores: {os.cpu_count()}")
    
    # สร้าง test data
    chunks = [list(range(i * 1000, (i+1) * 1000)) for i in range(20)]
    
    # Sequential
    start = time.perf_counter()
    seq_results = [cpu_intensive_task(chunk) for chunk in chunks]
    seq_time = time.perf_counter() - start
    
    # Parallel
    start = time.perf_counter()
    par_results = parallel_processing_pool(chunks)
    par_time = time.perf_counter() - start
    
    print(f"Sequential: {seq_time:.2f}s")
    print(f"Parallel ({os.cpu_count()} cores): {par_time:.2f}s")
    print(f"Speedup: {seq_time/par_time:.1f}x")
```

---

## 8. สรุป Part 099

✅ ใช้ asyncio อย่างถูกต้อง: parallel vs sequential, semaphore, TaskGroup
✅ ตั้งค่า Connection Pooling สำหรับ PostgreSQL, Redis และ HTTP clients
✅ แก้ N+1 Problem ด้วย Eager Loading (joinedload, selectinload)
✅ ใช้ Pagination ที่มีประสิทธิภาพ (Keyset Pagination)
✅ Profiling ด้วย cProfile, memory_profiler และ custom timing
✅ Memory optimization ด้วย `__slots__`, generators, weakref
✅ แนะนำ Cython และ NumPy สำหรับ high-performance computations
✅ ใช้ Multiprocessing Pool สำหรับ CPU-intensive tasks

## ➡️ ถัดไป: Part 100 - Distributed Systems

*Part 099/105 | Python Course - World-Class Level*
