# Part 100: Distributed Systems

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ CAP Theorem และ tradeoffs ใน distributed systems
- ออกแบบ Distributed Transactions ด้วย 2PC และ Saga
- เข้าใจ Consensus Algorithms พื้นฐาน (Raft/Paxos)
- ใช้ Distributed Caching อย่างมีประสิทธิภาพ
- เข้าใจ Service Mesh concepts
- Implement Observability (Metrics, Logs, Traces)

---

## 1. CAP Theorem

```python
# cap_theorem/concepts.py
"""
CAP Theorem: ระบบ Distributed ไม่สามารถมีคุณสมบัติ 3 อย่างพร้อมกันได้

C - Consistency: ทุก node เห็นข้อมูลเหมือนกันในทุกเวลา
A - Availability: ทุก request ได้รับ response (ไม่ error)
P - Partition Tolerance: ระบบทำงานต่อได้แม้ network แบ่งออกจากกัน

เลือกได้ 2 จาก 3:
- CP: Consistent + Partition Tolerant (ยอมเสีย Availability)
  - HBase, MongoDB (w:majority), ZooKeeper
  - เหมาะกับ: financial transactions, inventory

- AP: Available + Partition Tolerant (ยอมเสีย Consistency)
  - Cassandra, DynamoDB, CouchDB
  - เหมาะกับ: social media likes, shopping cart

- CA: Consistent + Available (ไม่ tolerant ต่อ partition)
  - ใช้ได้เฉพาะ single-node หรือ single datacenter
  - PostgreSQL, MySQL (ไม่ใช่ distributed)

Modern: PACELC Theorem ขยาย CAP ด้วย Latency
"""

# ตัวอย่าง: ระบบ inventory ที่เลือก Consistency over Availability
class ConsistentInventory:
    """
    CP System: ยอมเสีย Availability เพื่อรักษา Consistency
    ใช้สำหรับ inventory management ที่ต้องแม่นยำ
    """
    
    def __init__(self):
        self.stock = {}
        self._locks = {}  # Distributed locks
    
    def reserve_stock(self, product_id: int, quantity: int) -> bool:
        """
        Reserve stock พร้อม distributed lock
        CP: ถ้า lock ไม่ได้ = unavailable แต่ data consistent
        """
        import threading
        
        lock = self._locks.setdefault(product_id, threading.Lock())
        
        if not lock.acquire(timeout=5.0):  # timeout = availability tradeoff
            # ในระบบ distributed จริง: raise ServiceUnavailableError
            return False
        
        try:
            current_stock = self.stock.get(product_id, 0)
            if current_stock < quantity:
                return False
            
            self.stock[product_id] = current_stock - quantity
            return True
        finally:
            lock.release()


# ตัวอย่าง: Shopping cart ที่เลือก Availability over Consistency
class EventuallyConsistentCart:
    """
    AP System: ยอมเสีย Consistency เพื่อรักษา Availability
    ใช้สำหรับ shopping cart (Eventual Consistency)
    """
    
    def __init__(self, redis_client):
        self.redis = redis_client
    
    def add_item(self, user_id: int, product_id: int, quantity: int):
        """
        Add item ไปยัง cart
        AP: ตอบทันที ไม่รอ synchronization ทุก nodes
        """
        import json
        
        cart_key = f"cart:{user_id}"
        cart_data = self.redis.get(cart_key)
        
        cart = json.loads(cart_data) if cart_data else {}
        
        if str(product_id) in cart:
            cart[str(product_id)] += quantity
        else:
            cart[str(product_id)] = quantity
        
        # Write ทันที ไม่รอ replica sync
        self.redis.setex(cart_key, 86400, json.dumps(cart))
        
        # Background sync ไปยัง other regions (eventual consistency)
        # asyncio.create_task(self._sync_to_regions(user_id, cart))
        
        return cart


# Eventual Consistency Pattern
class EventuallyConsistentCounter:
    """
    Counter ที่ใช้ CRDT (Conflict-Free Replicated Data Type)
    ทุก node increment แยกกัน แล้ว merge ทีหลัง
    """
    
    def __init__(self, node_id: str, redis_client):
        self.node_id = node_id
        self.redis = redis_client
    
    def increment(self, counter_name: str, value: int = 1):
        """Increment counter ใน local node"""
        key = f"counter:{counter_name}:{self.node_id}"
        self.redis.incrby(key, value)
    
    def get_total(self, counter_name: str) -> int:
        """รวม counter จากทุก nodes"""
        pattern = f"counter:{counter_name}:*"
        keys = self.redis.keys(pattern)
        
        total = 0
        for key in keys:
            value = self.redis.get(key)
            if value:
                total += int(value)
        
        return total
```

---

## 2. Distributed Transactions (2PC)

```python
# distributed/two_phase_commit.py
"""
Two-Phase Commit (2PC) Protocol

Phase 1 - Prepare:
  Coordinator ถาม participants: "คุณพร้อม commit ได้ไหม?"
  Participant ตอบ: Yes หรือ No

Phase 2 - Commit/Abort:
  ถ้าทุก participants ตอบ Yes: Coordinator ส่ง Commit
  ถ้ามี participant ตอบ No: Coordinator ส่ง Abort

ข้อเสีย: Blocking protocol - ถ้า coordinator crash ใน phase 2
"""
import asyncio
import httpx
import logging
from typing import List, Dict, Optional
from enum import Enum
import uuid

logger = logging.getLogger(__name__)


class TransactionState(Enum):
    PREPARING = "preparing"
    PREPARED = "prepared"
    COMMITTING = "committing"
    COMMITTED = "committed"
    ABORTING = "aborting"
    ABORTED = "aborted"


class TwoPhaseCommitCoordinator:
    """2PC Coordinator"""
    
    def __init__(self, participants: List[str]):
        self.participants = participants
        self.transactions: Dict[str, dict] = {}
    
    async def execute_transaction(self, operations: Dict[str, dict]) -> bool:
        """
        Execute distributed transaction
        
        Args:
            operations: {participant_url: operation_data}
        """
        transaction_id = str(uuid.uuid4())
        
        self.transactions[transaction_id] = {
            "state": TransactionState.PREPARING,
            "participants": list(operations.keys()),
            "votes": {}
        }
        
        logger.info(f"Starting 2PC transaction: {transaction_id}")
        
        # Phase 1: Prepare
        phase1_success = await self._phase1_prepare(transaction_id, operations)
        
        if phase1_success:
            # Phase 2: Commit
            success = await self._phase2_commit(transaction_id)
        else:
            # Phase 2: Abort
            success = await self._phase2_abort(transaction_id)
        
        return success
    
    async def _phase1_prepare(
        self,
        transaction_id: str,
        operations: Dict[str, dict]
    ) -> bool:
        """Phase 1: ส่ง prepare ไปยังทุก participants"""
        async with httpx.AsyncClient(timeout=10.0) as client:
            prepare_tasks = []
            
            for participant_url, operation in operations.items():
                task = client.post(
                    f"{participant_url}/prepare",
                    json={
                        "transaction_id": transaction_id,
                        "operation": operation
                    }
                )
                prepare_tasks.append((participant_url, task))
            
            # รอทุก participants ตอบ
            all_prepared = True
            for participant_url, task in prepare_tasks:
                try:
                    response = await task
                    vote = response.json().get("vote") == "yes"
                    self.transactions[transaction_id]["votes"][participant_url] = vote
                    
                    if not vote:
                        all_prepared = False
                        logger.warning(f"Participant {participant_url} voted NO")
                
                except Exception as e:
                    logger.error(f"Phase 1 failed for {participant_url}: {e}")
                    self.transactions[transaction_id]["votes"][participant_url] = False
                    all_prepared = False
        
        if all_prepared:
            self.transactions[transaction_id]["state"] = TransactionState.PREPARED
        
        return all_prepared
    
    async def _phase2_commit(self, transaction_id: str) -> bool:
        """Phase 2: Commit ไปยังทุก participants"""
        self.transactions[transaction_id]["state"] = TransactionState.COMMITTING
        participants = self.transactions[transaction_id]["participants"]
        
        async with httpx.AsyncClient(timeout=30.0) as client:
            commit_tasks = [
                client.post(
                    f"{p}/commit",
                    json={"transaction_id": transaction_id}
                )
                for p in participants
            ]
            
            results = await asyncio.gather(*commit_tasks, return_exceptions=True)
        
        all_committed = all(
            not isinstance(r, Exception) and r.status_code == 200
            for r in results
        )
        
        if all_committed:
            self.transactions[transaction_id]["state"] = TransactionState.COMMITTED
            logger.info(f"Transaction {transaction_id} committed successfully")
        else:
            # Handle partial commit (ยาก - ต้องมี compensating transactions)
            logger.error(f"Transaction {transaction_id} partially committed!")
            self.transactions[transaction_id]["state"] = TransactionState.ABORTED
        
        return all_committed
    
    async def _phase2_abort(self, transaction_id: str) -> bool:
        """Phase 2: Abort ไปยังทุก participants"""
        self.transactions[transaction_id]["state"] = TransactionState.ABORTING
        participants = self.transactions[transaction_id]["participants"]
        
        async with httpx.AsyncClient(timeout=30.0) as client:
            abort_tasks = [
                client.post(
                    f"{p}/abort",
                    json={"transaction_id": transaction_id}
                )
                for p in participants
            ]
            
            await asyncio.gather(*abort_tasks, return_exceptions=True)
        
        self.transactions[transaction_id]["state"] = TransactionState.ABORTED
        logger.info(f"Transaction {transaction_id} aborted")
        return False
```

---

## 3. Consensus Algorithm Basics (Raft)

```python
# consensus/raft_basics.py
"""
Raft Consensus Algorithm - Simplified

สร้างขึ้นเพื่อให้เข้าใจง่ายกว่า Paxos

States: Follower -> Candidate -> Leader

Leader Election:
1. Followers รอ heartbeat จาก Leader
2. ถ้าไม่ได้รับ heartbeat = election timeout
3. Follower เปลี่ยนเป็น Candidate
4. ส่ง RequestVote ไปยังทุก nodes
5. ถ้าได้ majority votes = เป็น Leader

Log Replication:
1. Client ส่ง command ไปยัง Leader
2. Leader append ไปยัง log
3. Leader ส่ง AppendEntries ไปยัง Followers
4. เมื่อ majority commit = respond ให้ Client
"""
import asyncio
import random
import time
from enum import Enum
from dataclasses import dataclass, field
from typing import List, Optional, Dict


class RaftState(Enum):
    FOLLOWER = "follower"
    CANDIDATE = "candidate"
    LEADER = "leader"


@dataclass
class LogEntry:
    term: int
    index: int
    command: dict


@dataclass
class RaftNode:
    """Simplified Raft node"""
    node_id: str
    peers: List[str] = field(default_factory=list)
    
    # Persistent state
    current_term: int = 0
    voted_for: Optional[str] = None
    log: List[LogEntry] = field(default_factory=list)
    
    # Volatile state
    commit_index: int = 0
    last_applied: int = 0
    state: RaftState = RaftState.FOLLOWER
    
    # Leader state
    next_index: Dict[str, int] = field(default_factory=dict)
    match_index: Dict[str, int] = field(default_factory=dict)
    
    # Election
    votes_received: int = 0
    leader_id: Optional[str] = None
    last_heartbeat: float = field(default_factory=time.time)
    
    def _election_timeout(self) -> float:
        """Random election timeout (150-300ms ตาม Raft paper)"""
        return random.uniform(0.150, 0.300)
    
    def become_candidate(self):
        """เปลี่ยนเป็น Candidate เพื่อเริ่ม election"""
        self.state = RaftState.CANDIDATE
        self.current_term += 1
        self.voted_for = self.node_id  # Vote for self
        self.votes_received = 1  # Count own vote
        print(f"[{self.node_id}] Became CANDIDATE for term {self.current_term}")
    
    def become_leader(self):
        """เปลี่ยนเป็น Leader"""
        self.state = RaftState.LEADER
        self.leader_id = self.node_id
        
        # Initialize leader-specific state
        for peer in self.peers:
            self.next_index[peer] = len(self.log) + 1
            self.match_index[peer] = 0
        
        print(f"[{self.node_id}] ✅ Became LEADER for term {self.current_term}")
    
    def become_follower(self, term: int, leader_id: str = None):
        """เปลี่ยนเป็น Follower"""
        self.state = RaftState.FOLLOWER
        self.current_term = term
        self.voted_for = None
        self.leader_id = leader_id
        self.last_heartbeat = time.time()
        print(f"[{self.node_id}] Became FOLLOWER for term {term}")
    
    def handle_vote_request(
        self,
        candidate_id: str,
        candidate_term: int,
        last_log_index: int,
        last_log_term: int
    ) -> dict:
        """ตอบรับหรือปฏิเสธ vote request"""
        
        # อัพเดท term ถ้าเจอ term ที่สูงกว่า
        if candidate_term > self.current_term:
            self.become_follower(candidate_term)
        
        # ปฏิเสธถ้า term ต่ำกว่า
        if candidate_term < self.current_term:
            return {"vote_granted": False, "term": self.current_term}
        
        # ตรวจสอบว่า log ของ candidate เป็น up-to-date
        my_last_log_term = self.log[-1].term if self.log else 0
        my_last_log_index = len(self.log)
        
        log_ok = (
            last_log_term > my_last_log_term or
            (last_log_term == my_last_log_term and last_log_index >= my_last_log_index)
        )
        
        # Vote ถ้ายังไม่ได้ vote หรือ vote ให้ candidate นี้แล้ว
        if (self.voted_for is None or self.voted_for == candidate_id) and log_ok:
            self.voted_for = candidate_id
            return {"vote_granted": True, "term": self.current_term}
        
        return {"vote_granted": False, "term": self.current_term}
    
    def append_entries(
        self,
        leader_term: int,
        leader_id: str,
        entries: List[LogEntry] = None
    ) -> dict:
        """รับ AppendEntries จาก Leader (heartbeat หรือ log replication)"""
        
        # อัพเดท term ถ้าเจอ term ที่สูงกว่า
        if leader_term > self.current_term:
            self.become_follower(leader_term, leader_id)
        elif leader_term == self.current_term:
            self.last_heartbeat = time.time()
            self.leader_id = leader_id
            if self.state == RaftState.CANDIDATE:
                self.become_follower(leader_term, leader_id)
        
        if leader_term < self.current_term:
            return {"success": False, "term": self.current_term}
        
        # Append entries ถ้ามี
        if entries:
            self.log.extend(entries)
            print(f"[{self.node_id}] Appended {len(entries)} entries")
        
        return {"success": True, "term": self.current_term}
```

---

## 4. Distributed Caching

```python
# distributed_cache/redis_cluster.py
"""
Distributed Caching ด้วย Redis Cluster

Redis Cluster:
- แบ่งข้อมูลเป็น 16384 hash slots
- แต่ละ node รับผิดชอบ slots บางส่วน
- Automatic failover เมื่อ node ล่ม
"""
from redis.cluster import RedisCluster, ClusterNode
from redis import Redis
import json
import hashlib
import time
from typing import Any, Optional, List


class DistributedCache:
    """Distributed cache ด้วย Redis Cluster"""
    
    def __init__(self, cluster_nodes: List[dict]):
        """
        Args:
            cluster_nodes: [{"host": "redis1", "port": 7000}, ...]
        """
        startup_nodes = [
            ClusterNode(node["host"], node["port"])
            for node in cluster_nodes
        ]
        
        self.cluster = RedisCluster(
            startup_nodes=startup_nodes,
            decode_responses=True,
            skip_full_coverage_check=True,
            socket_timeout=5,
            socket_connect_timeout=5
        )
    
    def get(self, key: str) -> Optional[Any]:
        """ดึงข้อมูลจาก cache"""
        data = self.cluster.get(key)
        if data:
            return json.loads(data)
        return None
    
    def set(self, key: str, value: Any, ttl: int = 300) -> bool:
        """เก็บข้อมูลใน cache"""
        serialized = json.dumps(value, default=str)
        return self.cluster.setex(key, ttl, serialized)
    
    def delete(self, *keys: str) -> int:
        """ลบ keys จาก cache"""
        return self.cluster.delete(*keys)
    
    def mget(self, keys: List[str]) -> dict:
        """ดึงหลาย keys พร้อมกัน"""
        values = self.cluster.mget(keys)
        result = {}
        for key, value in zip(keys, values):
            if value:
                result[key] = json.loads(value)
        return result
    
    def mset(self, mapping: dict, ttl: int = 300) -> bool:
        """เก็บหลาย keys พร้อมกัน"""
        pipe = self.cluster.pipeline()
        for key, value in mapping.items():
            pipe.setex(key, ttl, json.dumps(value, default=str))
        pipe.execute()
        return True


# Cache-aside pattern กับ Redis Cluster
class CacheAsideRepository:
    """Repository pattern พร้อม cache-aside"""
    
    def __init__(self, cache: DistributedCache, db_session):
        self.cache = cache
        self.db = db_session
    
    async def get_product(self, product_id: int) -> Optional[dict]:
        """ดึง product พร้อม cache-aside"""
        cache_key = f"product:{product_id}"
        
        # 1. Check cache
        cached = self.cache.get(cache_key)
        if cached:
            return cached
        
        # 2. Cache miss - fetch from DB
        product = await self.db.get_product(product_id)
        if not product:
            return None
        
        # 3. Store in cache
        self.cache.set(cache_key, product, ttl=300)
        return product
    
    async def update_product(self, product_id: int, data: dict) -> dict:
        """อัพเดท product และ invalidate cache"""
        # Update DB
        product = await self.db.update_product(product_id, data)
        
        # Invalidate cache
        self.cache.delete(f"product:{product_id}")
        
        # Invalidate related caches
        self.cache.delete(f"category:{product.get('category')}:products")
        
        return product
    
    async def get_products_by_category(self, category: str) -> List[dict]:
        """ดึง products ตาม category พร้อม cache"""
        cache_key = f"category:{category}:products"
        
        cached = self.cache.get(cache_key)
        if cached:
            return cached
        
        products = await self.db.get_products_by_category(category)
        
        # Cache ด้วย TTL สั้นกว่า (data เปลี่ยนบ่อย)
        self.cache.set(cache_key, products, ttl=60)
        return products


# Write-through cache
class WriteThroughCache:
    """Write-through: เขียน cache และ DB พร้อมกัน"""
    
    def __init__(self, cache: DistributedCache, db_session):
        self.cache = cache
        self.db = db_session
    
    async def set_user_preference(self, user_id: int, key: str, value: Any):
        """บันทึก user preference ทั้ง cache และ DB"""
        # Write to DB first
        await self.db.set_preference(user_id, key, value)
        
        # Write to cache
        cache_key = f"user:{user_id}:pref:{key}"
        self.cache.set(cache_key, value, ttl=3600)
```

---

## 5. Service Mesh Concepts

```python
# service_mesh/concepts.py
"""
Service Mesh คือ infrastructure layer สำหรับ service-to-service communication

Components:
1. Data Plane: Sidecar proxies (Envoy) ที่ handle traffic
2. Control Plane: จัดการ configuration (Istio, Linkerd)

Features:
- Load Balancing
- Service Discovery
- Traffic Management
- Mutual TLS (mTLS)
- Distributed Tracing
- Circuit Breaking
- Retry Policies

ตัวอย่าง: Istio VirtualService configuration
"""

# ตัวอย่าง Istio configurations (YAML)
ISTIO_VIRTUAL_SERVICE = """
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: product-service
spec:
  hosts:
  - product-service
  http:
  - match:
    - headers:
        x-user-type:
          exact: "premium"
    route:
    - destination:
        host: product-service
        subset: v2
      weight: 100
  - route:
    - destination:
        host: product-service
        subset: v1
      weight: 90
    - destination:
        host: product-service
        subset: v2
      weight: 10
    # Canary deployment: 10% ไปยัง v2
"""

ISTIO_DESTINATION_RULE = """
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: product-service
spec:
  host: product-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    outlierDetection:  # Circuit breaking
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
"""

# Python implementation ของ service mesh concepts
class TrafficManager:
    """จำลอง traffic management ใน service mesh"""
    
    def __init__(self):
        self.routes = []
        self.circuit_breakers = {}
    
    def add_weighted_route(
        self,
        service: str,
        version: str,
        weight: int,
        conditions: dict = None
    ):
        """เพิ่ม weighted route"""
        self.routes.append({
            "service": service,
            "version": version,
            "weight": weight,
            "conditions": conditions or {}
        })
    
    def select_destination(self, request_headers: dict, service: str) -> str:
        """เลือก destination ตาม routing rules"""
        import random
        
        matching_routes = []
        for route in self.routes:
            if route["service"] != service:
                continue
            
            # ตรวจสอบ conditions
            conditions_met = all(
                request_headers.get(k) == v
                for k, v in route.get("conditions", {}).items()
            )
            
            if conditions_met:
                matching_routes.append(route)
        
        if not matching_routes:
            return f"{service}:v1"  # default
        
        # Weighted random selection
        total_weight = sum(r["weight"] for r in matching_routes)
        rand = random.uniform(0, total_weight)
        
        cumulative = 0
        for route in matching_routes:
            cumulative += route["weight"]
            if rand <= cumulative:
                return f"{service}:{route['version']}"
        
        return f"{service}:{matching_routes[-1]['version']}"
```

---

## 6. Observability

```python
# observability/metrics.py
"""
Observability Pillars:
1. Metrics - ตัวเลขวัด system health
2. Logs - events ที่เกิดขึ้น
3. Traces - การติดตาม request ข้าม services
"""
from prometheus_client import (
    Counter, Histogram, Gauge, Summary,
    start_http_server, REGISTRY
)
import time
import functools
from fastapi import FastAPI, Request, Response
import logging

# Custom Metrics
REQUEST_COUNT = Counter(
    "http_requests_total",
    "Total HTTP requests",
    labelnames=["method", "endpoint", "status_code"]
)

REQUEST_LATENCY = Histogram(
    "http_request_duration_seconds",
    "HTTP request latency",
    labelnames=["method", "endpoint"],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
)

ACTIVE_REQUESTS = Gauge(
    "http_active_requests",
    "Number of active HTTP requests",
    labelnames=["endpoint"]
)

DB_QUERY_DURATION = Histogram(
    "db_query_duration_seconds",
    "Database query duration",
    labelnames=["query_type", "table"]
)

CACHE_HIT_RATIO = Counter(
    "cache_operations_total",
    "Cache hit/miss counts",
    labelnames=["operation", "result"]  # operation: get, result: hit/miss
)

ERROR_COUNT = Counter(
    "application_errors_total",
    "Total application errors",
    labelnames=["type", "service"]
)


class MetricsMiddleware:
    """FastAPI Middleware สำหรับ collect metrics"""
    
    async def __call__(self, request: Request, call_next) -> Response:
        endpoint = request.url.path
        method = request.method
        
        # เพิ่ม active request gauge
        ACTIVE_REQUESTS.labels(endpoint=endpoint).inc()
        
        start_time = time.perf_counter()
        
        try:
            response = await call_next(request)
            
            # บันทึก metrics
            elapsed = time.perf_counter() - start_time
            
            REQUEST_COUNT.labels(
                method=method,
                endpoint=endpoint,
                status_code=response.status_code
            ).inc()
            
            REQUEST_LATENCY.labels(
                method=method,
                endpoint=endpoint
            ).observe(elapsed)
            
            return response
        
        except Exception as e:
            ERROR_COUNT.labels(
                type=type(e).__name__,
                service="api"
            ).inc()
            raise
        
        finally:
            ACTIVE_REQUESTS.labels(endpoint=endpoint).dec()


# observability/logging.py
import structlog
import logging
import sys
import json
from datetime import datetime


def configure_structured_logging():
    """ตั้งค่า structured logging ด้วย structlog"""
    
    structlog.configure(
        processors=[
            structlog.stdlib.filter_by_level,
            structlog.stdlib.add_logger_name,
            structlog.stdlib.add_log_level,
            structlog.stdlib.PositionalArgumentsFormatter(),
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            structlog.processors.UnicodeDecoder(),
            structlog.processors.JSONRenderer()  # Output as JSON
        ],
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
        wrapper_class=structlog.stdlib.BoundLogger,
        cache_logger_on_first_use=True,
    )
    
    # ตั้งค่า root logger
    logging.basicConfig(
        format="%(message)s",
        stream=sys.stdout,
        level=logging.INFO,
    )


# ใช้ structured logger
logger = structlog.get_logger()


def log_request(func):
    """Decorator สำหรับ log requests"""
    @functools.wraps(func)
    async def wrapper(request: Request, *args, **kwargs):
        request_id = request.headers.get("X-Request-ID", "unknown")
        
        bound_logger = logger.bind(
            request_id=request_id,
            method=request.method,
            path=request.url.path,
            client_ip=request.client.host
        )
        
        bound_logger.info("request_started")
        start_time = time.perf_counter()
        
        try:
            response = await func(request, *args, **kwargs)
            elapsed_ms = (time.perf_counter() - start_time) * 1000
            
            bound_logger.info(
                "request_completed",
                status_code=response.status_code,
                elapsed_ms=round(elapsed_ms, 2)
            )
            return response
        
        except Exception as e:
            elapsed_ms = (time.perf_counter() - start_time) * 1000
            bound_logger.error(
                "request_failed",
                error=str(e),
                error_type=type(e).__name__,
                elapsed_ms=round(elapsed_ms, 2),
                exc_info=True
            )
            raise
    
    return wrapper


# observability/tracing.py
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.instrumentation.redis import RedisInstrumentor
import os


def setup_tracing(service_name: str, otlp_endpoint: str = None):
    """ตั้งค่า OpenTelemetry distributed tracing"""
    
    # สร้าง TracerProvider
    tracer_provider = TracerProvider()
    
    # ตั้งค่า exporter (Jaeger/Zipkin/Tempo ผ่าน OTLP)
    otlp_endpoint = otlp_endpoint or os.getenv(
        "OTEL_EXPORTER_OTLP_ENDPOINT",
        "http://jaeger:4317"
    )
    
    exporter = OTLPSpanExporter(endpoint=otlp_endpoint, insecure=True)
    
    tracer_provider.add_span_processor(
        BatchSpanProcessor(exporter)
    )
    
    trace.set_tracer_provider(tracer_provider)
    
    return trace.get_tracer(service_name)


# Auto-instrumentation
def instrument_app(app: FastAPI):
    """Auto-instrument FastAPI app"""
    # FastAPI
    FastAPIInstrumentor.instrument_app(app)
    
    # SQLAlchemy
    SQLAlchemyInstrumentor().instrument()
    
    # Redis
    RedisInstrumentor().instrument()


# Manual tracing
tracer = trace.get_tracer("order-service")


async def process_order_with_tracing(order_id: str, user_id: int):
    """Process order พร้อม distributed tracing"""
    
    with tracer.start_as_current_span("process_order") as span:
        # เพิ่ม attributes
        span.set_attribute("order.id", order_id)
        span.set_attribute("user.id", user_id)
        
        # Sub-spans
        with tracer.start_as_current_span("validate_inventory"):
            # ตรวจสอบ inventory
            await asyncio.sleep(0.01)
            span.add_event("inventory_validated")
        
        with tracer.start_as_current_span("process_payment") as payment_span:
            try:
                await asyncio.sleep(0.05)
                payment_span.set_attribute("payment.method", "credit_card")
                span.add_event("payment_processed")
            except Exception as e:
                payment_span.record_exception(e)
                payment_span.set_status(trace.Status(trace.StatusCode.ERROR, str(e)))
                raise
        
        span.set_status(trace.Status(trace.StatusCode.OK))
        return {"order_id": order_id, "status": "processed"}
```

---

## 7. Health Checks และ Readiness

```python
# health/health_checker.py
from fastapi import FastAPI
from typing import Dict, Callable
import asyncio
import time

app = FastAPI()


class HealthChecker:
    """Comprehensive health checking"""
    
    def __init__(self):
        self.checks: Dict[str, Callable] = {}
        self.start_time = time.time()
    
    def add_check(self, name: str, check_func: Callable):
        """เพิ่ม health check"""
        self.checks[name] = check_func
    
    async def run_all_checks(self) -> dict:
        """รัน health checks ทั้งหมด"""
        results = {}
        overall_healthy = True
        
        check_tasks = {
            name: asyncio.create_task(self._run_check(name, func))
            for name, func in self.checks.items()
        }
        
        for name, task in check_tasks.items():
            result = await task
            results[name] = result
            if result["status"] != "healthy":
                overall_healthy = False
        
        return {
            "status": "healthy" if overall_healthy else "unhealthy",
            "uptime_seconds": int(time.time() - self.start_time),
            "checks": results,
            "timestamp": time.time()
        }
    
    async def _run_check(self, name: str, check_func: Callable) -> dict:
        """รัน check เดียว"""
        start = time.perf_counter()
        try:
            result = await check_func()
            elapsed = (time.perf_counter() - start) * 1000
            return {
                "status": "healthy",
                "response_time_ms": round(elapsed, 2),
                "details": result
            }
        except Exception as e:
            elapsed = (time.perf_counter() - start) * 1000
            return {
                "status": "unhealthy",
                "response_time_ms": round(elapsed, 2),
                "error": str(e)
            }


health_checker = HealthChecker()


# ลงทะเบียน health checks
async def check_database():
    """ตรวจสอบ database connection"""
    # await db.execute("SELECT 1")
    return {"connected": True, "pool_size": 10}


async def check_redis():
    """ตรวจสอบ Redis connection"""
    # await redis_client.ping()
    return {"connected": True}


async def check_disk_space():
    """ตรวจสอบ disk space"""
    import shutil
    total, used, free = shutil.disk_usage("/")
    free_gb = free // (1024**3)
    
    if free_gb < 1:
        raise Exception(f"Low disk space: {free_gb}GB remaining")
    
    return {"free_gb": free_gb, "total_gb": total // (1024**3)}


health_checker.add_check("database", check_database)
health_checker.add_check("redis", check_redis)
health_checker.add_check("disk", check_disk_space)


@app.get("/health")
async def health():
    """Liveness probe"""
    return {"status": "alive"}


@app.get("/ready")
async def readiness():
    """Readiness probe - ตรวจสอบว่า dependencies พร้อม"""
    result = await health_checker.run_all_checks()
    
    if result["status"] != "healthy":
        from fastapi import Response
        return Response(
            content=str(result),
            status_code=503
        )
    
    return result
```

---

## 8. สรุป Part 100

✅ เข้าใจ CAP Theorem และการเลือก CP vs AP ตาม use case
✅ Implement Two-Phase Commit (2PC) สำหรับ distributed transactions
✅ เข้าใจ Raft Consensus Algorithm พื้นฐาน (Leader Election, Log Replication)
✅ ใช้ Distributed Caching ด้วย Redis Cluster
✅ เข้าใจ Service Mesh concepts (Istio, traffic management)
✅ Implement Observability stack: Prometheus metrics, Structured logging, OpenTelemetry tracing
✅ สร้าง Health Checks สำหรับ Liveness และ Readiness probes

## ➡️ ถัดไป: Part 101 - Cloud Deployment (AWS/GCP)

*Part 100/105 | Python Course - World-Class Level*
