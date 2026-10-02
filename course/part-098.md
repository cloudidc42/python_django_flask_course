# Part 098: System Design for Python Apps

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- ออกแบบระบบ Horizontal Scaling สำหรับ Python apps
- ตั้งค่า Load Balancing ด้วย Nginx และ HAProxy
- ใช้ Database Sharding strategies
- Implement Caching strategies (CDN, Redis, In-memory)
- สร้าง Rate Limiting ระดับ production
- ใช้ Circuit Breaker pattern สำหรับ fault tolerance

---

## 1. Horizontal Scaling

```python
# scaling/stateless_app.py
"""
หลักการ Horizontal Scaling:
- ทำให้แอปเป็น Stateless (ไม่เก็บ state ใน server memory)
- ใช้ shared storage สำหรับ session, cache
- Design สำหรับ multiple instances

12-Factor App Principles ที่สำคัญ:
1. Codebase: one repo
2. Dependencies: explicit declare
3. Config: store in environment
4. Backing services: treat as attached resources
5. Stateless processes: ไม่เก็บ state ใน process
6. Port binding: expose service via port
"""

from fastapi import FastAPI, Request, Depends
from redis import Redis
import os
import socket
import uuid

app = FastAPI()

# ใช้ Redis แทน in-memory session storage
redis_client = Redis(
    host=os.getenv("REDIS_HOST", "redis"),
    port=int(os.getenv("REDIS_PORT", 6379)),
    decode_responses=True
)

# Instance ID สำหรับ debugging
INSTANCE_ID = os.getenv("INSTANCE_ID", socket.gethostname())


@app.get("/info")
async def get_instance_info():
    """แสดงข้อมูล instance ปัจจุบัน (สำหรับ load balancing verification)"""
    return {
        "instance_id": INSTANCE_ID,
        "hostname": socket.gethostname(),
        "pid": os.getpid()
    }


class SessionManager:
    """Session manager ที่ใช้ Redis (shared storage)"""
    
    def __init__(self, redis: Redis, session_ttl: int = 3600):
        self.redis = redis
        self.ttl = session_ttl
    
    def create_session(self, user_data: dict) -> str:
        """สร้าง session ใน Redis"""
        session_id = str(uuid.uuid4())
        import json
        self.redis.setex(
            f"session:{session_id}",
            self.ttl,
            json.dumps(user_data)
        )
        return session_id
    
    def get_session(self, session_id: str) -> dict | None:
        """ดึง session จาก Redis"""
        import json
        data = self.redis.get(f"session:{session_id}")
        if data:
            return json.loads(data)
        return None
    
    def update_session(self, session_id: str, data: dict):
        """อัพเดท session"""
        import json
        self.redis.setex(
            f"session:{session_id}",
            self.ttl,
            json.dumps(data)
        )
    
    def delete_session(self, session_id: str):
        """ลบ session"""
        self.redis.delete(f"session:{session_id}")
    
    def refresh_ttl(self, session_id: str):
        """ต่ออายุ session"""
        self.redis.expire(f"session:{session_id}", self.ttl)


session_manager = SessionManager(redis_client)
```

### Nginx Load Balancer Configuration

```nginx
# nginx/nginx.conf
upstream python_app {
    # Least connections algorithm - ส่งไปยัง server ที่มี connections น้อยที่สุด
    least_conn;
    
    server app1:8000 weight=2;  # server นี้รับงานมากกว่า
    server app2:8000 weight=1;
    server app3:8000 weight=1;
    
    # Health check
    keepalive 32;
}

# Rate limiting zone
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/m;
limit_req_zone $binary_remote_addr zone=auth_limit:10m rate=10r/m;

server {
    listen 80;
    server_name api.example.com;
    
    # Redirect HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name api.example.com;
    
    # SSL Configuration
    ssl_certificate /etc/ssl/certs/api.crt;
    ssl_certificate_key /etc/ssl/private/api.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
    
    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Strict-Transport-Security "max-age=31536000" always;
    
    # Gzip compression
    gzip on;
    gzip_types application/json text/plain text/css;
    gzip_min_length 1000;
    
    # API endpoints
    location /api/ {
        # Rate limiting
        limit_req zone=api_limit burst=20 nodelay;
        
        proxy_pass http://python_app;
        proxy_http_version 1.1;
        
        # Headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Connection "";
        
        # Timeouts
        proxy_connect_timeout 10s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
        
        # Buffering
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
    }
    
    # Auth endpoints - stricter rate limit
    location /api/auth/ {
        limit_req zone=auth_limit burst=5 nodelay;
        
        proxy_pass http://python_app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    
    # Static files served directly
    location /static/ {
        root /var/www;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
    
    # Health check endpoint (bypass rate limit)
    location /health {
        proxy_pass http://python_app;
        access_log off;
    }
}
```

---

## 2. Database Sharding

```python
# database/sharding.py
"""
Database Sharding strategies:

1. Horizontal Sharding (Range-based):
   - User IDs 1-1M -> Shard 1
   - User IDs 1M-2M -> Shard 2
   
2. Hash-based Sharding:
   - shard = hash(key) % num_shards
   
3. Directory-based Sharding:
   - Lookup table เก็บ mapping ว่าข้อมูลอยู่ที่ shard ไหน
"""
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session
from typing import Dict, Optional, List
import hashlib


class ShardManager:
    """จัดการ database shards"""
    
    def __init__(self, shard_configs: List[str]):
        """
        Args:
            shard_configs: list of database URLs for each shard
        """
        self.num_shards = len(shard_configs)
        self.engines: Dict[int, object] = {}
        self.session_factories: Dict[int, sessionmaker] = {}
        
        for shard_id, db_url in enumerate(shard_configs):
            engine = create_engine(
                db_url,
                pool_size=5,
                max_overflow=10,
                pool_pre_ping=True
            )
            self.engines[shard_id] = engine
            self.session_factories[shard_id] = sessionmaker(bind=engine)
    
    def get_shard_id(self, key: str | int) -> int:
        """คำนวณ shard ID จาก key"""
        if isinstance(key, int):
            return key % self.num_shards
        
        # Hash-based sharding สำหรับ string key
        hash_value = int(hashlib.md5(str(key).encode()).hexdigest(), 16)
        return hash_value % self.num_shards
    
    def get_session(self, key: str | int) -> Session:
        """ดึง database session สำหรับ key"""
        shard_id = self.get_shard_id(key)
        return self.session_factories[shard_id]()
    
    def get_all_sessions(self) -> List[Session]:
        """ดึง sessions ทุก shards (สำหรับ cross-shard queries)"""
        return [factory() for factory in self.session_factories.values()]


# ตัวอย่างการใช้ Sharding กับ User data
SHARD_CONFIGS = [
    "postgresql://user:pass@db-shard-0:5432/users",
    "postgresql://user:pass@db-shard-1:5432/users",
    "postgresql://user:pass@db-shard-2:5432/users",
    "postgresql://user:pass@db-shard-3:5432/users",
]

shard_manager = ShardManager(SHARD_CONFIGS)


class UserRepository:
    """Repository ที่รองรับ sharding"""
    
    def __init__(self, shard_manager: ShardManager):
        self.shards = shard_manager
    
    def create_user(self, user_data: dict) -> dict:
        """สร้าง user ใน shard ที่เหมาะสม"""
        # ใช้ email เป็น sharding key (consistent placement)
        email = user_data["email"]
        db = self.shards.get_session(email)
        
        try:
            # Insert user
            shard_id = self.shards.get_shard_id(email)
            print(f"Creating user in shard {shard_id}")
            
            # db.add(User(**user_data))
            # db.commit()
            return {**user_data, "shard_id": shard_id}
        finally:
            db.close()
    
    def get_user_by_id(self, user_id: int) -> Optional[dict]:
        """ดึง user จาก shard ด้วย user_id"""
        db = self.shards.get_session(user_id)
        try:
            # user = db.query(User).filter(User.id == user_id).first()
            # return user
            return {"user_id": user_id, "shard": self.shards.get_shard_id(user_id)}
        finally:
            db.close()
    
    def search_users(self, query: str) -> List[dict]:
        """ค้นหา users ทุก shards (scatter-gather)"""
        results = []
        sessions = self.shards.get_all_sessions()
        
        # ค้นหาพร้อมกันทุก shards
        import concurrent.futures
        
        def search_shard(session: Session) -> List[dict]:
            try:
                # users = session.query(User).filter(User.name.ilike(f"%{query}%")).all()
                # return [user.to_dict() for user in users]
                return []
            finally:
                session.close()
        
        with concurrent.futures.ThreadPoolExecutor() as executor:
            futures = [executor.submit(search_shard, session) for session in sessions]
            for future in concurrent.futures.as_completed(futures):
                results.extend(future.result())
        
        return results


# Read Replicas (Master-Slave)
class ReadWriteDatabase:
    """Database ที่แยก read และ write operations"""
    
    def __init__(self, master_url: str, replica_urls: List[str]):
        self.master = create_engine(
            master_url,
            pool_size=5,
            pool_pre_ping=True
        )
        
        self.replicas = [
            create_engine(url, pool_size=10, pool_pre_ping=True)
            for url in replica_urls
        ]
        
        self._replica_index = 0
    
    def get_write_session(self) -> Session:
        """Session สำหรับ write operations (ไปที่ master)"""
        return sessionmaker(bind=self.master)()
    
    def get_read_session(self) -> Session:
        """Session สำหรับ read operations (ไปที่ replica)"""
        if not self.replicas:
            return self.get_write_session()
        
        # Round-robin ระหว่าง replicas
        replica = self.replicas[self._replica_index % len(self.replicas)]
        self._replica_index += 1
        return sessionmaker(bind=replica)()
```

---

## 3. Caching Strategies

```python
# caching/cache_manager.py
"""
Caching Strategies:

1. Cache-Aside (Lazy Loading)
2. Write-Through Cache
3. Write-Behind (Write-Back) Cache
4. Read-Through Cache
"""
import redis
import json
import functools
import hashlib
import time
from typing import Any, Optional, Callable, TypeVar
import os

T = TypeVar("T")

redis_client = redis.Redis(
    host=os.getenv("REDIS_HOST", "localhost"),
    port=6379,
    decode_responses=True
)


class CacheManager:
    """จัดการ cache ด้วย Redis"""
    
    def __init__(self, client: redis.Redis, default_ttl: int = 300):
        self.client = client
        self.default_ttl = default_ttl
    
    def get(self, key: str) -> Optional[Any]:
        """ดึงข้อมูลจาก cache"""
        data = self.client.get(key)
        if data:
            return json.loads(data)
        return None
    
    def set(self, key: str, value: Any, ttl: int = None) -> bool:
        """เก็บข้อมูลใน cache"""
        serialized = json.dumps(value, default=str)
        return self.client.setex(key, ttl or self.default_ttl, serialized)
    
    def delete(self, key: str) -> int:
        """ลบข้อมูลจาก cache"""
        return self.client.delete(key)
    
    def delete_pattern(self, pattern: str) -> int:
        """ลบ keys ที่ match pattern"""
        keys = self.client.keys(pattern)
        if keys:
            return self.client.delete(*keys)
        return 0
    
    def get_or_set(self, key: str, fetch_func: Callable, ttl: int = None) -> Any:
        """Cache-Aside pattern: ดึงจาก cache หรือ fetch และ cache"""
        cached = self.get(key)
        if cached is not None:
            return cached
        
        # Cache miss - fetch จาก database
        value = fetch_func()
        if value is not None:
            self.set(key, value, ttl)
        return value
    
    def invalidate_related(self, entity_type: str, entity_id: int):
        """ลบ cache ที่เกี่ยวข้องกับ entity"""
        patterns = [
            f"{entity_type}:{entity_id}",
            f"{entity_type}:{entity_id}:*",
            f"list:{entity_type}:*",  # invalidate list caches
        ]
        for pattern in patterns:
            self.delete_pattern(pattern)


# Decorator สำหรับ cache
def cached(ttl: int = 300, key_prefix: str = ""):
    """Decorator สำหรับ cache function results"""
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        async def async_wrapper(*args, **kwargs):
            # สร้าง cache key จาก function name และ arguments
            key_parts = [key_prefix or func.__name__]
            key_parts.extend(str(a) for a in args)
            key_parts.extend(f"{k}:{v}" for k, v in sorted(kwargs.items()))
            cache_key = ":".join(key_parts)
            
            # ตรวจสอบ cache
            cached_value = cache_manager.get(cache_key)
            if cached_value is not None:
                return cached_value
            
            # Execute function
            result = await func(*args, **kwargs)
            
            # เก็บใน cache
            if result is not None:
                cache_manager.set(cache_key, result, ttl)
            
            return result
        
        @functools.wraps(func)
        def sync_wrapper(*args, **kwargs):
            key_parts = [key_prefix or func.__name__]
            key_parts.extend(str(a) for a in args)
            key_parts.extend(f"{k}:{v}" for k, v in sorted(kwargs.items()))
            cache_key = ":".join(key_parts)
            
            cached_value = cache_manager.get(cache_key)
            if cached_value is not None:
                return cached_value
            
            result = func(*args, **kwargs)
            if result is not None:
                cache_manager.set(cache_key, result, ttl)
            
            return result
        
        import asyncio
        if asyncio.iscoroutinefunction(func):
            return async_wrapper
        return sync_wrapper
    
    return decorator


cache_manager = CacheManager(redis_client)


# การใช้งาน caching ใน FastAPI
from fastapi import FastAPI
from sqlalchemy.orm import Session

app = FastAPI()


@cached(ttl=600, key_prefix="product")
async def get_product_cached(product_id: int) -> dict:
    """ดึง product พร้อม caching"""
    # จำลองการดึงข้อมูลจาก database
    return {"id": product_id, "name": "Product Name", "price": 299.99}


# Multi-level caching
class MultiLevelCache:
    """
    Multi-level cache:
    Level 1: In-memory (process-local, เร็วที่สุด)
    Level 2: Redis (shared across instances)
    Level 3: Database (source of truth)
    """
    
    def __init__(self, redis_client: redis.Redis, l1_max_size: int = 1000):
        self.l1_cache: dict = {}  # In-memory cache
        self.l1_ttls: dict = {}
        self.l1_max_size = l1_max_size
        self.redis = redis_client
    
    def get(self, key: str) -> Optional[Any]:
        # L1: Check in-memory
        if key in self.l1_cache:
            if time.time() < self.l1_ttls.get(key, 0):
                return self.l1_cache[key]
            else:
                # Expired
                del self.l1_cache[key]
                del self.l1_ttls[key]
        
        # L2: Check Redis
        data = self.redis.get(key)
        if data:
            value = json.loads(data)
            # Populate L1
            self._set_l1(key, value, ttl=60)  # L1 TTL สั้นกว่า
            return value
        
        return None
    
    def set(self, key: str, value: Any, ttl: int = 300):
        """เก็บใน L1 และ L2"""
        self._set_l1(key, value, ttl=min(ttl, 60))  # L1 max 60s
        self.redis.setex(key, ttl, json.dumps(value, default=str))
    
    def _set_l1(self, key: str, value: Any, ttl: int = 60):
        """เก็บใน L1 cache"""
        # ลบ oldest entry ถ้า cache เต็ม
        if len(self.l1_cache) >= self.l1_max_size:
            oldest_key = min(self.l1_ttls, key=self.l1_ttls.get)
            del self.l1_cache[oldest_key]
            del self.l1_ttls[oldest_key]
        
        self.l1_cache[key] = value
        self.l1_ttls[key] = time.time() + ttl
    
    def invalidate(self, key: str):
        """ลบจากทุก levels"""
        self.l1_cache.pop(key, None)
        self.l1_ttls.pop(key, None)
        self.redis.delete(key)


# CDN Cache Headers
class CDNCacheMiddleware:
    """Middleware สำหรับตั้งค่า CDN cache headers"""
    
    CACHE_RULES = {
        "/api/products": {"max_age": 300, "stale_while_revalidate": 60},
        "/api/categories": {"max_age": 3600, "stale_while_revalidate": 300},
        "/static": {"max_age": 31536000, "immutable": True},  # 1 ปี
    }
    
    async def __call__(self, request, call_next):
        from fastapi import Request
        from fastapi.responses import Response
        
        response = await call_next(request)
        
        path = request.url.path
        
        # ค้นหา cache rule ที่ match
        for pattern, rules in self.CACHE_RULES.items():
            if path.startswith(pattern):
                cache_control_parts = [f"public", f"max-age={rules['max_age']}"]
                
                if rules.get("stale_while_revalidate"):
                    cache_control_parts.append(
                        f"stale-while-revalidate={rules['stale_while_revalidate']}"
                    )
                
                if rules.get("immutable"):
                    cache_control_parts.append("immutable")
                
                response.headers["Cache-Control"] = ", ".join(cache_control_parts)
                break
        else:
            # Default: ไม่ cache API responses
            if path.startswith("/api/"):
                response.headers["Cache-Control"] = "no-store"
        
        return response
```

---

## 4. Rate Limiting

```python
# rate_limiting/rate_limiter.py
"""
Rate Limiting Strategies:
1. Fixed Window: นับ requests ใน time window คงที่
2. Sliding Window: time window แบบ sliding
3. Token Bucket: refill tokens ที่อัตราคงที่
4. Leaky Bucket: process requests ที่อัตราคงที่
"""
import redis
import time
from typing import Optional, Tuple
from fastapi import Request, HTTPException
from functools import wraps


class RateLimiter:
    """Rate limiter ด้วย Redis"""
    
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
    
    def fixed_window(
        self,
        key: str,
        max_requests: int,
        window_seconds: int
    ) -> Tuple[bool, dict]:
        """
        Fixed Window Rate Limiting
        ง่าย แต่มีปัญหา burst ที่ขอบ window
        """
        window_key = f"rate:fixed:{key}:{int(time.time() / window_seconds)}"
        
        current = self.redis.incr(window_key)
        if current == 1:
            self.redis.expire(window_key, window_seconds)
        
        allowed = current <= max_requests
        remaining = max(0, max_requests - current)
        
        return allowed, {
            "limit": max_requests,
            "remaining": remaining,
            "reset": int(time.time() / window_seconds + 1) * window_seconds
        }
    
    def sliding_window(
        self,
        key: str,
        max_requests: int,
        window_seconds: int
    ) -> Tuple[bool, dict]:
        """
        Sliding Window Rate Limiting ด้วย Redis Sorted Set
        แม่นยำกว่า fixed window
        """
        now = time.time()
        window_start = now - window_seconds
        
        pipe = self.redis.pipeline()
        
        # ลบ entries ที่หมดอายุ
        pipe.zremrangebyscore(f"rate:sliding:{key}", 0, window_start)
        
        # นับ requests ใน window ปัจจุบัน
        pipe.zcard(f"rate:sliding:{key}")
        
        # เพิ่ม request ปัจจุบัน
        pipe.zadd(f"rate:sliding:{key}", {str(now): now})
        
        # ตั้ง expiry
        pipe.expire(f"rate:sliding:{key}", int(window_seconds) + 1)
        
        _, count, _, _ = pipe.execute()
        
        allowed = count < max_requests
        remaining = max(0, max_requests - count - (1 if allowed else 0))
        
        return allowed, {
            "limit": max_requests,
            "remaining": remaining,
            "reset": int(now + window_seconds)
        }
    
    def token_bucket(
        self,
        key: str,
        capacity: int,
        refill_rate: float,  # tokens per second
        tokens_per_request: int = 1
    ) -> Tuple[bool, dict]:
        """
        Token Bucket Rate Limiting
        อนุญาต burst requests ได้
        """
        bucket_key = f"rate:token:{key}"
        now = time.time()
        
        # Lua script สำหรับ atomic operation
        lua_script = """
        local key = KEYS[1]
        local capacity = tonumber(ARGV[1])
        local refill_rate = tonumber(ARGV[2])
        local tokens_requested = tonumber(ARGV[3])
        local now = tonumber(ARGV[4])
        
        local bucket = redis.call('hmget', key, 'tokens', 'last_refill')
        local tokens = tonumber(bucket[1]) or capacity
        local last_refill = tonumber(bucket[2]) or now
        
        -- คำนวณ tokens ที่เพิ่มมาตั้งแต่ครั้งล่าสุด
        local elapsed = now - last_refill
        local new_tokens = tokens + (elapsed * refill_rate)
        if new_tokens > capacity then
            new_tokens = capacity
        end
        
        if new_tokens >= tokens_requested then
            new_tokens = new_tokens - tokens_requested
            redis.call('hmset', key, 'tokens', new_tokens, 'last_refill', now)
            redis.call('expire', key, math.ceil(capacity / refill_rate) + 1)
            return {1, math.floor(new_tokens)}
        else
            redis.call('hmset', key, 'tokens', new_tokens, 'last_refill', now)
            redis.call('expire', key, math.ceil(capacity / refill_rate) + 1)
            return {0, math.floor(new_tokens)}
        end
        """
        
        result = self.redis.eval(
            lua_script, 1, bucket_key,
            capacity, refill_rate, tokens_per_request, now
        )
        
        allowed = bool(result[0])
        remaining_tokens = int(result[1])
        
        return allowed, {
            "limit": capacity,
            "remaining": remaining_tokens,
            "retry_after": (tokens_per_request / refill_rate) if not allowed else None
        }


# Rate Limiting Middleware สำหรับ FastAPI
class RateLimitMiddleware:
    """Middleware สำหรับ rate limiting"""
    
    LIMITS = {
        "default": {"max_requests": 100, "window": 60},
        "/api/auth/login": {"max_requests": 5, "window": 60},
        "/api/auth/register": {"max_requests": 3, "window": 3600},
        "/api/search": {"max_requests": 30, "window": 60},
    }
    
    def __init__(self, redis_client: redis.Redis):
        self.limiter = RateLimiter(redis_client)
    
    async def __call__(self, request: Request, call_next):
        # ระบุ client
        client_ip = request.client.host
        api_key = request.headers.get("X-API-Key")
        client_id = api_key or client_ip
        
        # หา limit rule
        path = request.url.path
        limit_config = self.LIMITS.get(path, self.LIMITS["default"])
        
        # ตรวจสอบ rate limit
        key = f"{client_id}:{path}"
        allowed, info = self.limiter.sliding_window(
            key=key,
            max_requests=limit_config["max_requests"],
            window_seconds=limit_config["window"]
        )
        
        if not allowed:
            from fastapi.responses import JSONResponse
            return JSONResponse(
                status_code=429,
                content={
                    "error": "Rate limit exceeded",
                    "retry_after": limit_config["window"]
                },
                headers={
                    "X-RateLimit-Limit": str(info["limit"]),
                    "X-RateLimit-Remaining": "0",
                    "X-RateLimit-Reset": str(info["reset"]),
                    "Retry-After": str(limit_config["window"])
                }
            )
        
        response = await call_next(request)
        
        # เพิ่ม rate limit headers
        response.headers["X-RateLimit-Limit"] = str(info["limit"])
        response.headers["X-RateLimit-Remaining"] = str(info["remaining"])
        response.headers["X-RateLimit-Reset"] = str(info["reset"])
        
        return response


# Per-user rate limiting (authenticated)
def rate_limit(max_requests: int = 100, window: int = 60, per: str = "ip"):
    """Decorator สำหรับ rate limit endpoints"""
    def decorator(func):
        @wraps(func)
        async def wrapper(request: Request, *args, **kwargs):
            if per == "ip":
                client_id = request.client.host
            elif per == "user":
                # ดึง user ID จาก JWT token
                token = request.headers.get("Authorization", "").replace("Bearer ", "")
                client_id = f"user:{token[:10]}" if token else request.client.host
            else:
                client_id = request.client.host
            
            key = f"rate:{func.__name__}:{client_id}"
            r = redis.Redis(host="localhost", decode_responses=True)
            limiter = RateLimiter(r)
            
            allowed, info = limiter.sliding_window(key, max_requests, window)
            
            if not allowed:
                raise HTTPException(
                    status_code=429,
                    detail="Rate limit exceeded",
                    headers={"Retry-After": str(window)}
                )
            
            return await func(request, *args, **kwargs)
        
        return wrapper
    return decorator
```

---

## 5. Circuit Breaker Pattern

```python
# patterns/circuit_breaker_advanced.py
"""
Circuit Breaker States:
1. CLOSED: ทำงานปกติ
2. OPEN: หยุดส่ง requests ชั่วคราว
3. HALF_OPEN: ลองส่ง limited requests
"""
import asyncio
import time
import functools
import logging
from enum import Enum
from typing import Callable, Optional, TypeVar, Any
from dataclasses import dataclass, field
from collections import deque
import redis
import json
import os

logger = logging.getLogger(__name__)
T = TypeVar("T")


@dataclass
class CircuitBreakerConfig:
    """Configuration สำหรับ Circuit Breaker"""
    failure_threshold: int = 5          # จำนวน failure ก่อน OPEN
    success_threshold: int = 2          # จำนวน success ก่อน CLOSED จาก HALF_OPEN
    timeout: float = 60.0               # เวลาที่ OPEN ก่อนลอง HALF_OPEN
    half_open_max_calls: int = 3        # จำนวน calls สูงสุดใน HALF_OPEN
    failure_rate_threshold: float = 0.5  # % failure ก่อน OPEN
    min_calls: int = 10                 # calls ขั้นต่ำก่อนคำนวณ failure rate
    window_size: int = 60              # window สำหรับนับ failure rate (วินาที)


class CircuitState(Enum):
    CLOSED = "CLOSED"
    OPEN = "OPEN"
    HALF_OPEN = "HALF_OPEN"


class CircuitBreakerMetrics:
    """เก็บ metrics สำหรับ circuit breaker"""
    
    def __init__(self, window_size: int = 60):
        self.window_size = window_size
        self.calls: deque = deque()  # (timestamp, success: bool)
    
    def record_call(self, success: bool):
        now = time.time()
        self.calls.append((now, success))
        self._cleanup()
    
    def _cleanup(self):
        """ลบ calls ที่หมดอายุ"""
        cutoff = time.time() - self.window_size
        while self.calls and self.calls[0][0] < cutoff:
            self.calls.popleft()
    
    @property
    def total_calls(self) -> int:
        self._cleanup()
        return len(self.calls)
    
    @property
    def failure_rate(self) -> float:
        self._cleanup()
        if not self.calls:
            return 0.0
        failures = sum(1 for _, success in self.calls if not success)
        return failures / len(self.calls)
    
    @property
    def recent_failures(self) -> int:
        self._cleanup()
        return sum(1 for _, success in self.calls if not success)


class DistributedCircuitBreaker:
    """
    Circuit Breaker ที่ใช้ Redis สำหรับ distributed state
    ทุก instances ของ service ใช้ state เดียวกัน
    """
    
    def __init__(
        self,
        name: str,
        config: CircuitBreakerConfig,
        redis_client: redis.Redis
    ):
        self.name = name
        self.config = config
        self.redis = redis_client
        self.metrics = CircuitBreakerMetrics(config.window_size)
        self._state_key = f"cb:{name}:state"
        self._open_time_key = f"cb:{name}:open_time"
        self._half_open_calls_key = f"cb:{name}:half_open_calls"
    
    @property
    def state(self) -> CircuitState:
        """ดึง state จาก Redis"""
        state_str = self.redis.get(self._state_key)
        if state_str:
            return CircuitState(state_str)
        return CircuitState.CLOSED
    
    @state.setter
    def state(self, new_state: CircuitState):
        """บันทึก state ใน Redis"""
        self.redis.set(self._state_key, new_state.value)
        if new_state == CircuitState.OPEN:
            self.redis.set(self._open_time_key, str(time.time()))
        elif new_state == CircuitState.HALF_OPEN:
            self.redis.set(self._half_open_calls_key, "0")
    
    def _should_attempt(self) -> bool:
        """ตรวจสอบว่าควร attempt request หรือไม่"""
        current_state = self.state
        
        if current_state == CircuitState.CLOSED:
            return True
        
        if current_state == CircuitState.OPEN:
            open_time = float(self.redis.get(self._open_time_key) or 0)
            if time.time() - open_time > self.config.timeout:
                # เปลี่ยนเป็น HALF_OPEN
                self.state = CircuitState.HALF_OPEN
                logger.info(f"Circuit breaker [{self.name}]: OPEN -> HALF_OPEN")
                return True
            return False
        
        if current_state == CircuitState.HALF_OPEN:
            calls = int(self.redis.get(self._half_open_calls_key) or 0)
            return calls < self.config.half_open_max_calls
        
        return False
    
    def _on_success(self):
        current_state = self.state
        self.metrics.record_call(True)
        
        if current_state == CircuitState.HALF_OPEN:
            calls = self.redis.incr(self._half_open_calls_key)
            successes = int(calls)  # simplified - count as successes
            
            if successes >= self.config.success_threshold:
                self.state = CircuitState.CLOSED
                logger.info(f"Circuit breaker [{self.name}]: HALF_OPEN -> CLOSED")
    
    def _on_failure(self):
        current_state = self.state
        self.metrics.record_call(False)
        
        if current_state == CircuitState.HALF_OPEN:
            self.state = CircuitState.OPEN
            logger.warning(f"Circuit breaker [{self.name}]: HALF_OPEN -> OPEN (test failed)")
            return
        
        if current_state == CircuitState.CLOSED:
            should_open = False
            
            # ตรวจสอบ consecutive failures
            if self.metrics.recent_failures >= self.config.failure_threshold:
                should_open = True
            
            # ตรวจสอบ failure rate
            if (self.metrics.total_calls >= self.config.min_calls and
                    self.metrics.failure_rate >= self.config.failure_rate_threshold):
                should_open = True
            
            if should_open:
                self.state = CircuitState.OPEN
                logger.warning(
                    f"Circuit breaker [{self.name}]: CLOSED -> OPEN "
                    f"(failures: {self.metrics.recent_failures}, "
                    f"rate: {self.metrics.failure_rate:.2%})"
                )
    
    async def call(self, func: Callable, *args, fallback: Callable = None, **kwargs) -> Any:
        """Execute function ผ่าน circuit breaker"""
        if not self._should_attempt():
            logger.warning(f"Circuit breaker [{self.name}] is OPEN, rejecting request")
            
            if fallback:
                return await fallback(*args, **kwargs) if asyncio.iscoroutinefunction(fallback) else fallback(*args, **kwargs)
            
            raise Exception(f"Circuit breaker [{self.name}] is OPEN")
        
        try:
            if asyncio.iscoroutinefunction(func):
                result = await func(*args, **kwargs)
            else:
                result = func(*args, **kwargs)
            
            self._on_success()
            return result
        
        except Exception as e:
            self._on_failure()
            raise
    
    def get_stats(self) -> dict:
        """ดึง statistics ของ circuit breaker"""
        return {
            "name": self.name,
            "state": self.state.value,
            "total_calls": self.metrics.total_calls,
            "failure_rate": f"{self.metrics.failure_rate:.2%}",
            "recent_failures": self.metrics.recent_failures,
        }


# ตัวอย่างการใช้งาน
redis_cb = redis.Redis(host="localhost", decode_responses=True)

payment_cb = DistributedCircuitBreaker(
    name="payment-service",
    config=CircuitBreakerConfig(
        failure_threshold=5,
        timeout=30.0,
        failure_rate_threshold=0.5
    ),
    redis_client=redis_cb
)


async def charge_payment_with_cb(amount: float, user_id: int) -> dict:
    """เรียก payment service ผ่าน circuit breaker"""
    import httpx
    
    async def _charge():
        async with httpx.AsyncClient(timeout=5.0) as client:
            response = await client.post(
                "http://payment-service/charge",
                json={"amount": amount, "user_id": user_id}
            )
            response.raise_for_status()
            return response.json()
    
    async def _fallback(amount: float, user_id: int) -> dict:
        """Fallback: queue payment สำหรับ retry ภายหลัง"""
        logger.warning(f"Payment service unavailable, queuing payment for user {user_id}")
        # queue_payment_for_retry(amount, user_id)
        return {"status": "queued", "message": "Payment will be processed shortly"}
    
    return await payment_cb.call(_charge, fallback=_fallback, amount=amount, user_id=user_id)
```

---

## 6. สรุป Part 098

✅ ออกแบบ Stateless Applications สำหรับ Horizontal Scaling
✅ ตั้งค่า Nginx Load Balancer พร้อม upstream, SSL และ rate limiting
✅ ใช้ Database Sharding ทั้ง hash-based และ range-based
✅ Implement Multi-level Caching (in-memory + Redis + CDN headers)
✅ สร้าง Rate Limiter หลายแบบ (Fixed Window, Sliding Window, Token Bucket)
✅ ใช้ Distributed Circuit Breaker ด้วย Redis สำหรับ fault tolerance
✅ เข้าใจ Read/Write separation ด้วย Master-Replica databases

## ➡️ ถัดไป: Part 099 - High Performance Python

*Part 098/105 | Python Course - World-Class Level*
