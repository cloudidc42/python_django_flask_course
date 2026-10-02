# Part 102: Monitoring and Observability

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจหลักการ Observability และ Three Pillars: Metrics, Logs, Traces
- ใช้งาน Prometheus metrics ด้วย prometheus-client library
- สร้าง Custom Metrics: Counter, Gauge, Histogram, Summary
- Integrate Prometheus กับ FastAPI และ Django
- เข้าใจ Grafana dashboard concepts
- ใช้ structlog สำหรับ Structured Logging
- ตั้งค่า JSON logging format สำหรับ log aggregation
- Implement OpenTelemetry distributed tracing
- ใช้ Sentry SDK สำหรับ error tracking
- สร้าง Health check endpoints
- สร้าง Complete observability stack

---

## 1. ทำความเข้าใจ Observability

### Observability คืออะไร?

Observability คือความสามารถในการเข้าใจสถานะภายในของระบบจากผลลัพธ์ภายนอก ประกอบด้วย 3 เสาหลัก:

1. **Metrics** - ตัวเลขที่วัดประสิทธิภาพระบบ (CPU, Memory, Request rate)
2. **Logs** - บันทึกเหตุการณ์ที่เกิดขึ้นในระบบ
3. **Traces** - การติดตามเส้นทางของ request ผ่านระบบต่างๆ

```python
# ตัวอย่างแนวคิด: สิ่งที่ต้องวัดในระบบ web application
OBSERVABILITY_PILLARS = {
    "metrics": {
        "คำอธิบาย": "ตัวเลขที่เปลี่ยนแปลงตามเวลา",
        "ตัวอย่าง": [
            "requests_total",           # จำนวน request ทั้งหมด
            "request_duration_seconds", # เวลาในการตอบสนอง
            "active_connections",       # การเชื่อมต่อที่ active อยู่
            "memory_usage_bytes",       # การใช้ memory
        ]
    },
    "logs": {
        "คำอธิบาย": "บันทึกเหตุการณ์ที่มี context",
        "ตัวอย่าง": [
            "user_login event",
            "database_query executed",
            "error_occurred",
            "payment_processed",
        ]
    },
    "traces": {
        "คำอธิบาย": "การติดตาม request ผ่าน services ต่างๆ",
        "ตัวอย่าง": [
            "HTTP request → Auth service → DB → Cache → Response",
            "Order API → Inventory → Payment → Notification",
        ]
    }
}
```

### การติดตั้ง dependencies

```bash
# ติดตั้ง libraries ที่จำเป็น
pip install prometheus-client==0.20.0
pip install structlog==24.1.0
pip install opentelemetry-api==1.24.0
pip install opentelemetry-sdk==1.24.0
pip install opentelemetry-instrumentation-fastapi==0.45b0
pip install opentelemetry-exporter-otlp==1.24.0
pip install sentry-sdk[fastapi]==1.40.5
pip install fastapi==0.110.0
pip install uvicorn==0.27.1
pip install httpx==0.27.0

# สำหรับ Django
pip install django-prometheus==2.3.1
pip install django-structlog==8.0.0
```

---

## 2. Prometheus Metrics ด้วย prometheus-client

### ความรู้พื้นฐานเกี่ยวกับ Prometheus

Prometheus เป็น open-source monitoring system ที่ดึงข้อมูล (scrape) metrics จาก endpoints ที่เรากำหนด ทำงานแบบ pull-based

```python
# prometheus_basics.py
# พื้นฐานการสร้าง Prometheus metrics

from prometheus_client import (
    Counter,
    Gauge,
    Histogram,
    Summary,
    Info,
    Enum,
    start_http_server,
    REGISTRY,
    CollectorRegistry,
    generate_latest,
    CONTENT_TYPE_LATEST,
)
import time
import random
import threading

# ===== COUNTER: ตัวนับที่เพิ่มขึ้นเรื่อยๆ ไม่สามารถลดได้ =====
# ใช้สำหรับ: นับ requests, errors, events
http_requests_total = Counter(
    name='http_requests_total',
    documentation='จำนวน HTTP requests ทั้งหมด',
    labelnames=['method', 'endpoint', 'status_code']
)

# การใช้งาน Counter
def simulate_http_request(method: str, endpoint: str, status_code: int):
    """จำลองการนับ HTTP requests"""
    http_requests_total.labels(
        method=method,
        endpoint=endpoint,
        status_code=str(status_code)
    ).inc()  # เพิ่มค่า 1

# เพิ่มค่ามากกว่า 1
def process_batch(items_count: int):
    """นับ batch processing"""
    processed_items = Counter(
        'batch_items_processed_total',
        'จำนวน items ที่ประมวลผลแล้ว'
    )
    processed_items.inc(items_count)  # เพิ่ม n หน่วย


# ===== GAUGE: ค่าที่ขึ้นลงได้ ณ ช่วงเวลาหนึ่ง =====
# ใช้สำหรับ: จำนวน active connections, queue size, memory usage
active_connections = Gauge(
    name='active_connections',
    documentation='จำนวน connections ที่ active อยู่ตอนนี้'
)

queue_size = Gauge(
    name='queue_size',
    documentation='จำนวน items ใน queue',
    labelnames=['queue_name']
)

memory_usage_bytes = Gauge(
    name='process_memory_bytes',
    documentation='การใช้ memory ของ process ในหน่วย bytes'
)

# การใช้งาน Gauge
def simulate_connections():
    """จำลองการจัดการ connections"""
    active_connections.inc()    # เพิ่มขึ้น 1
    # ... ทำงาน ...
    active_connections.dec()    # ลดลง 1

def update_memory():
    """อัพเดทค่า memory"""
    import psutil
    memory_usage_bytes.set(
        psutil.Process().memory_info().rss  # กำหนดค่าโดยตรง
    )

# ใช้ Gauge เป็น context manager (track_inprogress)
def handle_request():
    """นับ requests ที่กำลังประมวลผลอยู่"""
    in_progress = Gauge('requests_in_progress', 'Requests กำลังประมวลผล')
    with in_progress.track_inprogress():
        # ทำงาน...
        time.sleep(0.1)


# ===== HISTOGRAM: กระจายของค่าในช่วง buckets =====
# ใช้สำหรับ: request duration, response size, latency distribution
request_duration_histogram = Histogram(
    name='http_request_duration_seconds',
    documentation='เวลาในการประมวลผล HTTP request',
    labelnames=['method', 'endpoint'],
    # กำหนด buckets: 1ms, 5ms, 10ms, 25ms, 50ms, 100ms, 250ms, 500ms, 1s, 2.5s, 5s
    buckets=[0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
)

# การใช้งาน Histogram
def time_request(method: str, endpoint: str, func):
    """วัดเวลา request ด้วย Histogram"""
    start_time = time.time()
    result = func()
    duration = time.time() - start_time
    
    request_duration_histogram.labels(
        method=method,
        endpoint=endpoint
    ).observe(duration)  # บันทึกค่าที่สังเกตได้
    
    return result

# ใช้ context manager
def time_with_histogram():
    """วัดเวลาด้วย context manager"""
    with request_duration_histogram.labels(
        method='GET',
        endpoint='/api/users'
    ).time():
        # ทำงาน...
        time.sleep(random.uniform(0.001, 0.5))


# ===== SUMMARY: คล้าย Histogram แต่คำนวณ quantiles ฝั่ง client =====
# ใช้สำหรับ: latency percentiles เมื่อต้องการ accuracy สูง
request_duration_summary = Summary(
    name='request_processing_seconds',
    documentation='เวลาในการประมวลผล request (summary)',
    labelnames=['endpoint'],
    quantiles=[0.5, 0.9, 0.95, 0.99, 0.999]  # p50, p90, p95, p99, p99.9
)

# การใช้งาน Summary
def measure_with_summary(endpoint: str):
    """วัดด้วย Summary"""
    with request_duration_summary.labels(endpoint=endpoint).time():
        time.sleep(random.uniform(0.001, 0.1))


# ===== INFO: ข้อมูล metadata ของ process =====
app_info = Info(
    name='app',
    documentation='ข้อมูล application'
)

# กำหนดข้อมูลครั้งเดียว
app_info.info({
    'version': '1.0.0',
    'build_date': '2024-01-15',
    'git_commit': 'abc123def',
    'python_version': '3.11.0',
    'environment': 'production'
})


# ===== ENUM: สถานะของ application =====
app_state = Enum(
    name='app_state',
    documentation='สถานะปัจจุบันของ application',
    states=['starting', 'running', 'degraded', 'stopping']
)

app_state.state('starting')
# ... หลังจาก startup เสร็จ ...
app_state.state('running')


# สาธิตการใช้งาน
if __name__ == '__main__':
    # เริ่ม HTTP server สำหรับ metrics endpoint (port 8000)
    start_http_server(8000)
    print("Metrics server started at http://localhost:8000/metrics")
    
    # จำลอง traffic
    while True:
        # สุ่ม request
        methods = ['GET', 'POST', 'PUT', 'DELETE']
        endpoints = ['/api/users', '/api/orders', '/api/products', '/health']
        status_codes = [200, 200, 200, 201, 400, 404, 500]  # 200 มีโอกาสสูงกว่า
        
        method = random.choice(methods)
        endpoint = random.choice(endpoints)
        status = random.choice(status_codes)
        
        # บันทึก metrics
        http_requests_total.labels(
            method=method,
            endpoint=endpoint,
            status_code=str(status)
        ).inc()
        
        # วัดเวลา
        with request_duration_histogram.labels(
            method=method,
            endpoint=endpoint
        ).time():
            time.sleep(random.uniform(0.001, 0.5))
        
        # อัพเดท active connections
        active_connections.set(random.randint(0, 100))
        
        time.sleep(0.1)  # หน่วง 100ms ระหว่าง requests
```

---

## 3. Custom Metrics แบบ Advanced

```python
# custom_metrics.py
# Custom metrics ขั้นสูงสำหรับ production systems

from prometheus_client import (
    Counter, Gauge, Histogram, Summary,
    CollectorRegistry, multiprocess,
    REGISTRY
)
from prometheus_client.metrics import MetricWrapperBase
from typing import Callable, Optional
import time
import functools
import asyncio

# ===== Business Metrics: วัด KPIs ทางธุรกิจ =====
class BusinessMetrics:
    """รวม business metrics ที่สำคัญไว้ที่เดียว"""
    
    def __init__(self, registry=REGISTRY):
        # Revenue metrics
        self.revenue_total = Counter(
            'business_revenue_total',
            'รายรับทั้งหมดในหน่วย Baht',
            ['payment_method', 'product_category'],
            registry=registry
        )
        
        # Order metrics
        self.orders_created = Counter(
            'business_orders_created_total',
            'จำนวน orders ที่สร้างทั้งหมด',
            ['order_type', 'channel'],
            registry=registry
        )
        
        self.orders_completed = Counter(
            'business_orders_completed_total',
            'จำนวน orders ที่เสร็จสมบูรณ์',
            ['order_type'],
            registry=registry
        )
        
        self.order_cancellation_rate = Gauge(
            'business_order_cancellation_rate',
            'อัตราการยกเลิก order (0-1)',
            registry=registry
        )
        
        # User metrics
        self.active_users = Gauge(
            'business_active_users',
            'จำนวน users ที่ active ใน 24 ชั่วโมง',
            registry=registry
        )
        
        self.new_user_registrations = Counter(
            'business_user_registrations_total',
            'จำนวน users ใหม่ที่ลงทะเบียน',
            ['registration_source'],
            registry=registry
        )
        
        # Performance metrics ทางธุรกิจ
        self.checkout_duration = Histogram(
            'business_checkout_duration_seconds',
            'เวลาในกระบวนการ checkout',
            buckets=[1, 5, 10, 30, 60, 120, 300]
        )
        
        self.cart_abandonment_count = Counter(
            'business_cart_abandonments_total',
            'จำนวนครั้งที่ผู้ใช้ละทิ้ง cart'
        )
    
    def record_sale(self, amount: float, payment_method: str, category: str):
        """บันทึกการขาย"""
        self.revenue_total.labels(
            payment_method=payment_method,
            product_category=category
        ).inc(amount)
    
    def record_order(self, order_type: str, channel: str):
        """บันทึก order ใหม่"""
        self.orders_created.labels(
            order_type=order_type,
            channel=channel
        ).inc()


# ===== Infrastructure Metrics =====
class InfrastructureMetrics:
    """metrics สำหรับ infrastructure"""
    
    def __init__(self):
        # Database metrics
        self.db_connections_active = Gauge(
            'db_connections_active',
            'จำนวน database connections ที่ active',
            ['database', 'pool_name']
        )
        
        self.db_query_duration = Histogram(
            'db_query_duration_seconds',
            'เวลาในการ execute database query',
            ['query_type', 'table'],
            buckets=[0.0001, 0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0, 5.0]
        )
        
        self.db_errors_total = Counter(
            'db_errors_total',
            'จำนวน database errors ทั้งหมด',
            ['error_type', 'database']
        )
        
        # Cache metrics
        self.cache_hits = Counter(
            'cache_hits_total',
            'จำนวน cache hits',
            ['cache_name']
        )
        
        self.cache_misses = Counter(
            'cache_misses_total',
            'จำนวน cache misses',
            ['cache_name']
        )
        
        self.cache_size_bytes = Gauge(
            'cache_size_bytes',
            'ขนาด cache ในหน่วย bytes',
            ['cache_name']
        )
        
        # External API metrics
        self.external_api_requests = Counter(
            'external_api_requests_total',
            'จำนวน requests ไปยัง external APIs',
            ['api_name', 'method', 'status']
        )
        
        self.external_api_duration = Histogram(
            'external_api_request_duration_seconds',
            'เวลาในการเรียก external API',
            ['api_name', 'method']
        )
    
    def record_db_query(self, query_type: str, table: str, duration: float):
        """บันทึก database query metrics"""
        self.db_query_duration.labels(
            query_type=query_type,
            table=table
        ).observe(duration)
    
    def record_cache_operation(self, cache_name: str, hit: bool):
        """บันทึก cache hit/miss"""
        if hit:
            self.cache_hits.labels(cache_name=cache_name).inc()
        else:
            self.cache_misses.labels(cache_name=cache_name).inc()
    
    def get_cache_hit_rate(self, cache_name: str) -> float:
        """คำนวณ cache hit rate"""
        hits = self.cache_hits.labels(cache_name=cache_name)._value.get()
        misses = self.cache_misses.labels(cache_name=cache_name)._value.get()
        total = hits + misses
        return hits / total if total > 0 else 0.0


# ===== Decorator-based Metrics =====
def track_execution_time(metric_name: str, labels: dict = None):
    """Decorator สำหรับวัดเวลาการทำงานของ function"""
    histogram = Histogram(
        metric_name,
        f'เวลาในการทำงานของ {metric_name}',
        list(labels.keys()) if labels else []
    )
    
    def decorator(func: Callable):
        @functools.wraps(func)
        def sync_wrapper(*args, **kwargs):
            label_values = labels or {}
            with histogram.labels(**label_values).time():
                return func(*args, **kwargs)
        
        @functools.wraps(func)
        async def async_wrapper(*args, **kwargs):
            label_values = labels or {}
            start = time.time()
            try:
                return await func(*args, **kwargs)
            finally:
                duration = time.time() - start
                histogram.labels(**label_values).observe(duration)
        
        if asyncio.iscoroutinefunction(func):
            return async_wrapper
        return sync_wrapper
    
    return decorator


def count_calls(metric_name: str, labels: dict = None):
    """Decorator สำหรับนับการเรียก function"""
    counter = Counter(
        metric_name,
        f'จำนวนครั้งที่เรียก {metric_name}',
        list(labels.keys()) if labels else []
    )
    
    def decorator(func: Callable):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            label_values = labels or {}
            counter.labels(**label_values).inc()
            return func(*args, **kwargs)
        return wrapper
    
    return decorator


# ตัวอย่างการใช้ decorator
@track_execution_time(
    'payment_processing_duration_seconds',
    labels={'payment_type': 'credit_card'}
)
def process_payment(amount: float, card_token: str):
    """ประมวลผลการชำระเงิน"""
    time.sleep(0.1)  # จำลองการทำงาน
    return {"status": "success", "transaction_id": "TXN001"}


@count_calls('email_notifications_sent_total', labels={'template': 'welcome'})
def send_welcome_email(user_email: str):
    """ส่ง welcome email"""
    print(f"ส่ง email ไปยัง {user_email}")
```

---

## 4. FastAPI Prometheus Integration

```python
# fastapi_prometheus.py
# การ integrate Prometheus กับ FastAPI

from fastapi import FastAPI, Request, Response
from prometheus_client import (
    Counter, Gauge, Histogram, Summary,
    REGISTRY, generate_latest, CONTENT_TYPE_LATEST
)
from prometheus_client.exposition import choose_encoder
import time
import asyncio
from contextlib import asynccontextmanager
from typing import Callable
import uvicorn

# ===== กำหนด Metrics =====
# Request metrics
REQUEST_COUNT = Counter(
    'fastapi_requests_total',
    'จำนวน HTTP requests ทั้งหมด',
    ['app_name', 'method', 'endpoint', 'http_status']
)

REQUEST_LATENCY = Histogram(
    'fastapi_request_latency_seconds',
    'Request latency ในหน่วยวินาที',
    ['app_name', 'method', 'endpoint'],
    buckets=[0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
)

REQUESTS_IN_PROGRESS = Gauge(
    'fastapi_requests_in_progress',
    'จำนวน requests ที่กำลังประมวลผลอยู่',
    ['app_name', 'method', 'endpoint']
)

REQUEST_SIZE = Histogram(
    'fastapi_request_size_bytes',
    'ขนาด request body ในหน่วย bytes',
    ['app_name', 'method', 'endpoint'],
    buckets=[100, 1000, 10000, 100000, 1000000]
)

RESPONSE_SIZE = Histogram(
    'fastapi_response_size_bytes',
    'ขนาด response body ในหน่วย bytes',
    ['app_name', 'method', 'endpoint'],
    buckets=[100, 1000, 10000, 100000, 1000000]
)

# Application-specific metrics
DB_POOL_SIZE = Gauge(
    'app_db_pool_size',
    'ขนาด database connection pool'
)

CACHE_HIT_RATE = Gauge(
    'app_cache_hit_rate',
    'Cache hit rate'
)


class PrometheusMiddleware:
    """Middleware สำหรับเก็บ Prometheus metrics"""
    
    def __init__(self, app: FastAPI, app_name: str = "fastapi_app"):
        self.app = app
        self.app_name = app_name
        self.kwargs = {}
    
    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return
        
        request = Request(scope, receive)
        method = request.method
        # ทำ path template แทน actual path เพื่อลด cardinality
        endpoint = self._get_path_template(scope)
        
        # นับ requests in progress
        REQUESTS_IN_PROGRESS.labels(
            app_name=self.app_name,
            method=method,
            endpoint=endpoint
        ).inc()
        
        # วัดเวลา
        start_time = time.time()
        status_code = 500  # default ถ้ามี error
        
        async def send_wrapper(message):
            nonlocal status_code
            if message["type"] == "http.response.start":
                status_code = message["status"]
                # วัดขนาด response headers
                headers = dict(message.get("headers", []))
                content_length = headers.get(b"content-length", b"0")
                
            await send(message)
        
        try:
            await self.app(scope, receive, send_wrapper)
        except Exception as e:
            status_code = 500
            raise e
        finally:
            # บันทึก metrics หลังจาก request เสร็จ
            duration = time.time() - start_time
            
            REQUEST_COUNT.labels(
                app_name=self.app_name,
                method=method,
                endpoint=endpoint,
                http_status=status_code
            ).inc()
            
            REQUEST_LATENCY.labels(
                app_name=self.app_name,
                method=method,
                endpoint=endpoint
            ).observe(duration)
            
            REQUESTS_IN_PROGRESS.labels(
                app_name=self.app_name,
                method=method,
                endpoint=endpoint
            ).dec()
    
    def _get_path_template(self, scope: dict) -> str:
        """ดึง path template แทน actual path"""
        # ใช้ route pattern ถ้าเป็นไปได้
        if "route" in scope:
            return scope["route"].path
        return scope.get("path", "unknown")


# ===== สร้าง FastAPI Application =====
@asynccontextmanager
async def lifespan(app: FastAPI):
    """Startup/Shutdown events"""
    print("🚀 Application starting...")
    # Startup: กำหนดค่า initial metrics
    DB_POOL_SIZE.set(20)
    CACHE_HIT_RATE.set(0.85)
    
    yield
    
    print("🛑 Application shutting down...")


app = FastAPI(
    title="Monitored FastAPI App",
    description="ตัวอย่าง FastAPI ที่มี Prometheus monitoring",
    lifespan=lifespan
)

# เพิ่ม Prometheus middleware
app.add_middleware(PrometheusMiddleware, app_name="ecommerce_api")


# ===== Metrics Endpoint =====
@app.get("/metrics")
async def metrics():
    """Prometheus metrics endpoint"""
    # เลือก encoder ตาม Accept header
    return Response(
        content=generate_latest(REGISTRY),
        media_type=CONTENT_TYPE_LATEST
    )


# ===== Health Check Endpoints =====
@app.get("/health")
async def health_check():
    """Basic health check"""
    return {
        "status": "healthy",
        "timestamp": time.time(),
        "version": "1.0.0"
    }


@app.get("/health/detailed")
async def detailed_health_check():
    """Detailed health check ที่ตรวจสอบ dependencies"""
    health_status = {
        "status": "healthy",
        "checks": {}
    }
    
    # ตรวจสอบ database
    db_healthy = await check_database_health()
    health_status["checks"]["database"] = {
        "status": "healthy" if db_healthy else "unhealthy",
        "response_time_ms": 5.2  # จำลอง
    }
    
    # ตรวจสอบ cache
    cache_healthy = await check_cache_health()
    health_status["checks"]["cache"] = {
        "status": "healthy" if cache_healthy else "unhealthy",
        "hit_rate": 0.85
    }
    
    # ตรวจสอบ external services
    health_status["checks"]["payment_gateway"] = {
        "status": "healthy",
        "last_successful_call": "2024-01-15T10:30:00Z"
    }
    
    # ถ้า check ใดล้มเหลว ให้ status เป็น degraded
    if not db_healthy or not cache_healthy:
        health_status["status"] = "degraded"
    
    return health_status


async def check_database_health() -> bool:
    """ตรวจสอบว่า database ใช้งานได้"""
    try:
        # จำลอง: SELECT 1
        await asyncio.sleep(0.005)
        return True
    except Exception:
        return False


async def check_cache_health() -> bool:
    """ตรวจสอบว่า Redis/cache ใช้งานได้"""
    try:
        # จำลอง: PING
        await asyncio.sleep(0.001)
        return True
    except Exception:
        return False


# ===== Business API Endpoints =====
@app.get("/api/users")
async def get_users():
    """ดึงรายการ users"""
    # จำลองการทำงาน
    await asyncio.sleep(0.02)
    return {"users": [{"id": 1, "name": "สมชาย"}, {"id": 2, "name": "สมหญิง"}]}


@app.post("/api/orders")
async def create_order(order: dict):
    """สร้าง order ใหม่"""
    await asyncio.sleep(0.05)
    return {"order_id": "ORD001", "status": "created"}


if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8080)
```

---

## 5. Django Prometheus Integration

```python
# django_prometheus_setup.py
# การ integrate Prometheus กับ Django

# settings.py - เพิ่ม django-prometheus
INSTALLED_APPS = [
    # ... apps อื่นๆ ...
    'django_prometheus',  # ต้องอยู่ก่อน django.contrib.staticfiles
    'django.contrib.admin',
    'django.contrib.auth',
    # ...
]

MIDDLEWARE = [
    'django_prometheus.middleware.PrometheusBeforeMiddleware',  # ต้องอยู่แรกสุด
    # ... middleware อื่นๆ ...
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    # ...
    'django_prometheus.middleware.PrometheusAfterMiddleware',  # ต้องอยู่ท้ายสุด
]

# Database monitoring
DATABASES = {
    'default': {
        'ENGINE': 'django_prometheus.db.backends.postgresql',  # แทนที่ django.db.backends.postgresql
        'NAME': 'mydb',
        'USER': 'myuser',
        'PASSWORD': 'mypassword',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}

# Cache monitoring
CACHES = {
    'default': {
        'BACKEND': 'django_prometheus.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://localhost:6379/1',
    }
}
```

```python
# django_custom_metrics.py
# Custom metrics สำหรับ Django application

from prometheus_client import Counter, Gauge, Histogram
from django.db.models.signals import post_save, post_delete
from django.dispatch import receiver
from django.contrib.auth.signals import user_logged_in, user_logged_out
from django.core.signals import request_started
import time

# กำหนด custom metrics
USER_LOGINS = Counter(
    'django_user_logins_total',
    'จำนวนครั้งที่ user login',
    ['user_type']
)

USER_REGISTRATIONS = Counter(
    'django_user_registrations_total',
    'จำนวน user registrations ใหม่'
)

ACTIVE_SESSIONS = Gauge(
    'django_active_sessions',
    'จำนวน active sessions'
)

MODEL_OPERATIONS = Counter(
    'django_model_operations_total',
    'จำนวน database model operations',
    ['model', 'operation']
)

DJANGO_ORM_QUERY_TIME = Histogram(
    'django_orm_query_seconds',
    'เวลาใน ORM queries',
    ['model', 'operation'],
    buckets=[0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0]
)


# ===== Signal handlers =====
@receiver(user_logged_in)
def track_user_login(sender, request, user, **kwargs):
    """บันทึก user login"""
    user_type = 'admin' if user.is_staff else 'regular'
    USER_LOGINS.labels(user_type=user_type).inc()


@receiver(user_logged_out)
def track_user_logout(sender, request, user, **kwargs):
    """ลด active sessions"""
    if ACTIVE_SESSIONS._value.get() > 0:
        ACTIVE_SESSIONS.dec()


# ===== Model Metrics Mixin =====
class MetricsMixin:
    """Mixin สำหรับเพิ่ม metrics ให้กับ Django models"""
    
    @classmethod
    def _get_model_name(cls) -> str:
        return cls.__name__.lower()
    
    def save(self, *args, **kwargs):
        """Override save เพื่อบันทึก metrics"""
        is_new = self.pk is None
        operation = 'create' if is_new else 'update'
        
        start_time = time.time()
        super().save(*args, **kwargs)
        duration = time.time() - start_time
        
        MODEL_OPERATIONS.labels(
            model=self._get_model_name(),
            operation=operation
        ).inc()
        
        DJANGO_ORM_QUERY_TIME.labels(
            model=self._get_model_name(),
            operation=operation
        ).observe(duration)
    
    def delete(self, *args, **kwargs):
        """Override delete เพื่อบันทึก metrics"""
        start_time = time.time()
        result = super().delete(*args, **kwargs)
        duration = time.time() - start_time
        
        MODEL_OPERATIONS.labels(
            model=self._get_model_name(),
            operation='delete'
        ).inc()
        
        return result


# ตัวอย่างการใช้ MetricsMixin กับ Django model
"""
# models.py
from django.db import models
from .metrics import MetricsMixin

class Order(MetricsMixin, models.Model):
    user = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    total_amount = models.DecimalField(max_digits=10, decimal_places=2)
    status = models.CharField(max_length=50)
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        db_table = 'orders'
"""

# urls.py - เพิ่ม metrics endpoint
"""
from django.urls import path, include

urlpatterns = [
    # ... URLs อื่นๆ ...
    path('', include('django_prometheus.urls')),  # เพิ่ม /metrics endpoint
]
"""
```

---

## 6. Grafana Dashboard Concepts

```python
# grafana_dashboard_config.py
# ตัวอย่าง Grafana dashboard configuration แบบ JSON

# นี่คือ conceptual code สำหรับเข้าใจโครงสร้าง Grafana dashboard
# ในการใช้งานจริง จะ configure ผ่าน Grafana UI หรือ Grafana API

GRAFANA_DASHBOARD_EXAMPLE = {
    "title": "Python Application Dashboard",
    "description": "Dashboard สำหรับ monitor Python web application",
    "tags": ["python", "fastapi", "production"],
    "timezone": "Asia/Bangkok",
    "refresh": "30s",  # auto refresh ทุก 30 วินาที
    
    "panels": [
        # ===== Row 1: Overview =====
        {
            "type": "stat",
            "title": "Total Requests (5m)",
            "description": "จำนวน requests ทั้งหมดในช่วง 5 นาทีที่ผ่านมา",
            "query": "sum(rate(fastapi_requests_total[5m]))",
            "unit": "reqps",  # requests per second
            "thresholds": [
                {"value": 0, "color": "green"},
                {"value": 100, "color": "yellow"},
                {"value": 500, "color": "red"}
            ]
        },
        {
            "type": "stat",
            "title": "Error Rate",
            "description": "อัตราส่วน 5xx errors",
            "query": """
                sum(rate(fastapi_requests_total{http_status=~"5.."}[5m])) /
                sum(rate(fastapi_requests_total[5m])) * 100
            """,
            "unit": "percent",
            "thresholds": [
                {"value": 0, "color": "green"},
                {"value": 1, "color": "yellow"},
                {"value": 5, "color": "red"}
            ]
        },
        {
            "type": "stat",
            "title": "p99 Latency",
            "description": "Latency ที่ percentile 99",
            "query": """
                histogram_quantile(0.99,
                    sum(rate(fastapi_request_latency_seconds_bucket[5m])) by (le)
                )
            """,
            "unit": "s"
        },
        
        # ===== Row 2: Request Rate Graph =====
        {
            "type": "graph",
            "title": "Request Rate by Endpoint",
            "description": "อัตรา requests แต่ละ endpoint",
            "query": """
                sum by (endpoint, method) (
                    rate(fastapi_requests_total[5m])
                )
            """,
            "legend": True,
            "yAxis": {"unit": "reqps"}
        },
        
        # ===== Row 3: Latency Distribution =====
        {
            "type": "heatmap",
            "title": "Request Latency Heatmap",
            "description": "การกระจายของ request latency",
            "query": """
                sum by (le) (
                    rate(fastapi_request_latency_seconds_bucket[1m])
                )
            """,
        },
        
        # ===== Row 4: Error Analysis =====
        {
            "type": "table",
            "title": "Top Error Endpoints",
            "description": "Endpoints ที่มี error มากที่สุด",
            "query": """
                topk(10,
                    sum by (endpoint, http_status) (
                        increase(fastapi_requests_total{http_status=~"4..|5.."}[1h])
                    )
                )
            """
        }
    ],
    
    # Alerting rules
    "alerts": [
        {
            "name": "High Error Rate",
            "condition": """
                sum(rate(fastapi_requests_total{http_status=~"5.."}[5m])) /
                sum(rate(fastapi_requests_total[5m])) > 0.05
            """,
            "severity": "critical",
            "message": "Error rate สูงกว่า 5% - กรุณาตรวจสอบระบบ",
            "channels": ["slack-alerts", "pagerduty"]
        },
        {
            "name": "High Latency",
            "condition": """
                histogram_quantile(0.99,
                    rate(fastapi_request_latency_seconds_bucket[5m])
                ) > 2.0
            """,
            "severity": "warning",
            "message": "p99 latency สูงกว่า 2 วินาที"
        },
        {
            "name": "Service Down",
            "condition": "up{job='fastapi_app'} == 0",
            "severity": "critical",
            "message": "Application ไม่สามารถ scrape metrics ได้ - อาจ down"
        }
    ]
}


# Prometheus scrape configuration (prometheus.yml)
PROMETHEUS_CONFIG = """
global:
  scrape_interval: 15s      # scrape metrics ทุก 15 วินาที
  evaluation_interval: 15s  # evaluate rules ทุก 15 วินาที
  scrape_timeout: 10s

scrape_configs:
  - job_name: 'fastapi_app'
    static_configs:
      - targets: ['app:8080']  # application host:port
    metrics_path: '/metrics'
    
  - job_name: 'django_app'
    static_configs:
      - targets: ['django:8000']
    metrics_path: '/metrics'
    
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['node-exporter:9100']
    # ดึง system metrics: CPU, memory, disk, network

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - 'rules/*.yml'  # ไฟล์ alerting rules
"""
```

---

## 7. Structured Logging ด้วย structlog

```python
# structured_logging.py
# Structured logging ด้วย structlog library

import structlog
import logging
import sys
import json
from datetime import datetime
from typing import Any
import uuid

# ===== ตั้งค่า structlog =====
def configure_structlog(
    environment: str = "production",
    log_level: str = "INFO",
    json_output: bool = True
):
    """ตั้งค่า structlog สำหรับ production"""
    
    # กำหนด shared processors ที่ใช้ร่วมกัน
    shared_processors = [
        # เพิ่ม timestamp
        structlog.processors.TimeStamper(fmt="iso"),
        # เพิ่ม log level
        structlog.stdlib.add_log_level,
        # เพิ่ม logger name
        structlog.stdlib.add_logger_name,
        # เพิ่ม call location (file, line)
        structlog.processors.CallsiteParameterAdder([
            structlog.processors.CallsiteParameter.FILENAME,
            structlog.processors.CallsiteParameter.LINENO,
            structlog.processors.CallsiteParameter.FUNC_NAME,
        ]),
        # รองรับ exception info
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
    ]
    
    if json_output:
        # Production: output เป็น JSON
        processors = shared_processors + [
            structlog.processors.dict_tracebacks,
            structlog.processors.JSONRenderer()
        ]
    else:
        # Development: output ที่อ่านได้ง่าย
        processors = shared_processors + [
            structlog.dev.ConsoleRenderer(colors=True)
        ]
    
    structlog.configure(
        processors=processors,
        wrapper_class=structlog.make_filtering_bound_logger(
            getattr(logging, log_level.upper())
        ),
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(sys.stdout),
        cache_logger_on_first_use=True,
    )
    
    # ตั้งค่า standard library logging ด้วย
    logging.basicConfig(
        format="%(message)s",
        stream=sys.stdout,
        level=getattr(logging, log_level.upper()),
    )


# เรียกใช้งานตอน startup
configure_structlog(environment="production", json_output=True)


# ===== การใช้งาน structlog =====
# สร้าง logger
logger = structlog.get_logger(__name__)


class RequestLogger:
    """Logger สำหรับ HTTP requests ที่มี context ครบ"""
    
    def __init__(self):
        self.logger = structlog.get_logger("request")
    
    def log_request(
        self,
        method: str,
        path: str,
        status_code: int,
        duration_ms: float,
        request_id: str,
        user_id: str = None,
        **kwargs
    ):
        """Log HTTP request พร้อม context"""
        log_method = self.logger.info if status_code < 400 else \
                     self.logger.warning if status_code < 500 else \
                     self.logger.error
        
        log_method(
            "http_request",                    # event name
            method=method,
            path=path,
            status_code=status_code,
            duration_ms=round(duration_ms, 2),
            request_id=request_id,
            user_id=user_id,
            **kwargs
        )
    
    def log_error(self, error: Exception, request_id: str, context: dict = None):
        """Log error พร้อม full context"""
        self.logger.error(
            "request_error",
            error_type=type(error).__name__,
            error_message=str(error),
            request_id=request_id,
            exc_info=True,
            **(context or {})
        )


class AuditLogger:
    """Logger สำหรับ audit trail - บันทึกการกระทำสำคัญ"""
    
    def __init__(self):
        self.logger = structlog.get_logger("audit")
    
    def log_user_action(
        self,
        user_id: str,
        action: str,
        resource_type: str,
        resource_id: str,
        changes: dict = None,
        ip_address: str = None
    ):
        """บันทึก user action สำหรับ audit"""
        self.logger.info(
            "user_action",
            user_id=user_id,
            action=action,                    # login, logout, create, update, delete
            resource_type=resource_type,      # order, product, user
            resource_id=resource_id,
            changes=changes,                  # before/after values
            ip_address=ip_address,
            audit_event_id=str(uuid.uuid4()), # unique event ID
            timestamp=datetime.utcnow().isoformat()
        )
    
    def log_security_event(
        self,
        event_type: str,
        severity: str,
        user_id: str = None,
        ip_address: str = None,
        details: dict = None
    ):
        """บันทึก security event"""
        log_method = {
            "low": self.logger.info,
            "medium": self.logger.warning,
            "high": self.logger.error,
            "critical": self.logger.critical
        }.get(severity, self.logger.warning)
        
        log_method(
            "security_event",
            event_type=event_type,     # failed_login, suspicious_activity, etc.
            severity=severity,
            user_id=user_id,
            ip_address=ip_address,
            **(details or {})
        )


# ===== Context-bound logging =====
def process_order(order_id: str, user_id: str):
    """ตัวอย่างการใช้ bound logger สำหรับ context"""
    
    # สร้าง logger ที่มี context ผูกไว้
    log = logger.bind(
        order_id=order_id,
        user_id=user_id,
        operation="process_order"
    )
    
    log.info("เริ่มประมวลผล order")
    
    try:
        # ตรวจสอบ inventory
        log.debug("กำลังตรวจสอบ inventory", items=["ITEM001", "ITEM002"])
        
        # ประมวลผลการชำระเงิน
        log.info("กำลังประมวลผลการชำระเงิน", amount=1500.00)
        
        # อัพเดทสถานะ
        log.info("อัพเดทสถานะ order", new_status="confirmed")
        
        return {"status": "success"}
        
    except Exception as e:
        log.error("เกิดข้อผิดพลาดในการประมวลผล order", error=str(e), exc_info=True)
        raise


# ตัวอย่าง output JSON log
EXAMPLE_JSON_LOG = {
    "timestamp": "2024-01-15T10:30:45.123456Z",
    "level": "info",
    "logger": "order_processor",
    "event": "เริ่มประมวลผล order",
    "order_id": "ORD-001",
    "user_id": "USR-123",
    "operation": "process_order",
    "filename": "order_service.py",
    "lineno": 45,
    "func_name": "process_order"
}
```

---

## 8. JSON Logging สำหรับ Log Aggregation

```python
# json_logging.py
# JSON logging สำหรับ ELK Stack, Loki, CloudWatch

import logging
import json
import sys
import traceback
from datetime import datetime, timezone
from typing import Any, Optional
import uuid

class JSONFormatter(logging.Formatter):
    """
    Custom JSON formatter สำหรับ structured logging
    Compatible กับ ELK Stack (Elasticsearch, Logstash, Kibana)
    """
    
    def __init__(
        self,
        app_name: str = "python_app",
        environment: str = "production",
        version: str = "1.0.0",
        extra_fields: dict = None
    ):
        super().__init__()
        self.app_name = app_name
        self.environment = environment
        self.version = version
        self.extra_fields = extra_fields or {}
    
    def format(self, record: logging.LogRecord) -> str:
        """แปลง log record เป็น JSON string"""
        
        # ข้อมูลพื้นฐาน
        log_entry = {
            # === ฟิลด์มาตรฐาน ===
            "@timestamp": datetime.now(timezone.utc).isoformat(),
            "level": record.levelname.lower(),
            "message": record.getMessage(),
            
            # === Application context ===
            "app": self.app_name,
            "environment": self.environment,
            "version": self.version,
            
            # === Source location ===
            "logger": record.name,
            "module": record.module,
            "function": record.funcName,
            "line": record.lineno,
            "thread": record.thread,
            "process": record.process,
        }
        
        # เพิ่ม extra fields ที่กำหนดไว้
        log_entry.update(self.extra_fields)
        
        # เพิ่ม extra fields จาก record
        if hasattr(record, 'extra'):
            log_entry.update(record.extra)
        
        # เพิ่ม request context ถ้ามี
        for field in ['request_id', 'user_id', 'session_id', 'trace_id', 'span_id']:
            if hasattr(record, field):
                log_entry[field] = getattr(record, field)
        
        # จัดการ exception
        if record.exc_info:
            log_entry['exception'] = {
                'type': record.exc_info[0].__name__ if record.exc_info[0] else None,
                'message': str(record.exc_info[1]),
                'stacktrace': self.formatException(record.exc_info)
            }
        
        # Stack trace ถ้ามี
        if record.stack_info:
            log_entry['stack_info'] = record.stack_info
        
        return json.dumps(log_entry, ensure_ascii=False, default=str)


class ContextualLogger:
    """
    Logger ที่รองรับ context สำหรับ request tracking
    """
    
    def __init__(self, name: str):
        self._logger = logging.getLogger(name)
        self._context = {}
    
    def bind(self, **kwargs) -> 'ContextualLogger':
        """สร้าง logger ใหม่ที่มี context ผูกไว้"""
        new_logger = ContextualLogger(self._logger.name)
        new_logger._context = {**self._context, **kwargs}
        return new_logger
    
    def _log(self, level: int, msg: str, **kwargs):
        """Log message พร้อม context"""
        extra = {**self._context, **kwargs}
        self._logger.log(level, msg, extra={'extra': extra})
    
    def debug(self, msg: str, **kwargs):
        self._log(logging.DEBUG, msg, **kwargs)
    
    def info(self, msg: str, **kwargs):
        self._log(logging.INFO, msg, **kwargs)
    
    def warning(self, msg: str, **kwargs):
        self._log(logging.WARNING, msg, **kwargs)
    
    def error(self, msg: str, **kwargs):
        self._log(logging.ERROR, msg, **kwargs)
    
    def critical(self, msg: str, **kwargs):
        self._log(logging.CRITICAL, msg, **kwargs)


def setup_json_logging(
    app_name: str,
    environment: str,
    log_level: str = "INFO",
    log_file: Optional[str] = None
):
    """ตั้งค่า JSON logging"""
    
    formatter = JSONFormatter(
        app_name=app_name,
        environment=environment
    )
    
    # Console handler
    console_handler = logging.StreamHandler(sys.stdout)
    console_handler.setFormatter(formatter)
    
    handlers = [console_handler]
    
    # File handler ถ้ากำหนด
    if log_file:
        file_handler = logging.FileHandler(log_file)
        file_handler.setFormatter(formatter)
        handlers.append(file_handler)
    
    # ตั้งค่า root logger
    logging.basicConfig(
        level=getattr(logging, log_level.upper()),
        handlers=handlers
    )
    
    # ปิด noise logs จาก libraries
    logging.getLogger("uvicorn.access").setLevel(logging.WARNING)
    logging.getLogger("sqlalchemy.engine").setLevel(logging.WARNING)


# ===== FastAPI JSON Logging Middleware =====
from fastapi import Request
import time

class JSONLoggingMiddleware:
    """Middleware สำหรับ log ทุก HTTP request เป็น JSON"""
    
    def __init__(self, app, app_name: str = "fastapi"):
        self.app = app
        self.logger = ContextualLogger(f"{app_name}.access")
    
    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return
        
        request = Request(scope, receive)
        request_id = str(uuid.uuid4())
        start_time = time.time()
        
        # Bind request context ให้ logger
        log = self.logger.bind(
            request_id=request_id,
            method=request.method,
            path=request.url.path,
            remote_addr=request.client.host if request.client else "unknown"
        )
        
        status_code = 500
        
        async def send_wrapper(message):
            nonlocal status_code
            if message["type"] == "http.response.start":
                status_code = message["status"]
            await send(message)
        
        try:
            await self.app(scope, receive, send_wrapper)
        finally:
            duration_ms = (time.time() - start_time) * 1000
            
            log_func = log.info if status_code < 400 else log.warning if status_code < 500 else log.error
            log_func(
                "http_request_completed",
                status_code=status_code,
                duration_ms=round(duration_ms, 2),
                user_agent=request.headers.get("user-agent", ""),
            )
```

---

## 9. OpenTelemetry Distributed Tracing

```python
# opentelemetry_tracing.py
# Distributed Tracing ด้วย OpenTelemetry

from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import (
    BatchSpanProcessor,
    ConsoleSpanExporter,
    SimpleSpanProcessor
)
from opentelemetry.sdk.resources import Resource, SERVICE_NAME, SERVICE_VERSION
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.trace import Status, StatusCode
from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator
from opentelemetry.propagate import inject, extract
import functools
from typing import Callable, Optional
import asyncio


def setup_telemetry(
    service_name: str,
    service_version: str = "1.0.0",
    environment: str = "production",
    otlp_endpoint: str = "http://otel-collector:4317",
    export_to_console: bool = False
):
    """ตั้งค่า OpenTelemetry tracing"""
    
    # กำหนด resource attributes (metadata ของ service)
    resource = Resource.create({
        SERVICE_NAME: service_name,
        SERVICE_VERSION: service_version,
        "deployment.environment": environment,
        "host.name": "app-server-01",
    })
    
    # สร้าง TracerProvider
    provider = TracerProvider(resource=resource)
    
    # ตั้งค่า exporter
    if export_to_console:
        # Development: export ไปที่ console
        provider.add_span_processor(
            SimpleSpanProcessor(ConsoleSpanExporter())
        )
    else:
        # Production: export ไปที่ OTLP collector (Jaeger, Tempo, etc.)
        otlp_exporter = OTLPSpanExporter(
            endpoint=otlp_endpoint,
            insecure=True  # ใช้ TLS ใน production จริงๆ
        )
        provider.add_span_processor(
            BatchSpanProcessor(
                otlp_exporter,
                max_queue_size=2048,
                max_export_batch_size=512,
                export_timeout_millis=30000,
            )
        )
    
    # Register global TracerProvider
    trace.set_tracer_provider(provider)
    
    return provider


def instrument_fastapi(app):
    """Instrument FastAPI application"""
    FastAPIInstrumentor.instrument_app(app)
    RequestsInstrumentor().instrument()


def get_tracer(name: str) -> trace.Tracer:
    """ดึง tracer สำหรับ module"""
    return trace.get_tracer(name)


# ===== Manual Tracing =====
tracer = get_tracer(__name__)


class OrderService:
    """Order service ที่มี distributed tracing"""
    
    async def create_order(self, user_id: str, items: list, total: float):
        """สร้าง order พร้อม tracing"""
        
        # สร้าง root span สำหรับ operation นี้
        with tracer.start_as_current_span("create_order") as span:
            # เพิ่ม attributes
            span.set_attribute("user.id", user_id)
            span.set_attribute("order.item_count", len(items))
            span.set_attribute("order.total", total)
            
            try:
                # Span สำหรับ validate order
                with tracer.start_as_current_span("validate_order") as validate_span:
                    validate_span.set_attribute("validation.items_count", len(items))
                    await self._validate_order(items)
                    validate_span.set_status(Status(StatusCode.OK))
                
                # Span สำหรับ check inventory
                with tracer.start_as_current_span("check_inventory") as inv_span:
                    for item in items:
                        inv_span.set_attribute(f"item.{item['id']}.quantity", item['quantity'])
                    
                    available = await self._check_inventory(items)
                    inv_span.set_attribute("inventory.all_available", available)
                    
                    if not available:
                        inv_span.set_status(Status(StatusCode.ERROR, "Insufficient inventory"))
                        raise ValueError("สินค้าบางรายการไม่มีในคลัง")
                
                # Span สำหรับ process payment
                with tracer.start_as_current_span("process_payment") as pay_span:
                    pay_span.set_attribute("payment.amount", total)
                    pay_span.set_attribute("payment.currency", "THB")
                    
                    payment_result = await self._process_payment(user_id, total)
                    pay_span.set_attribute("payment.transaction_id", payment_result['transaction_id'])
                    pay_span.set_status(Status(StatusCode.OK))
                
                # Span สำหรับ save to database
                with tracer.start_as_current_span("save_order") as db_span:
                    db_span.set_attribute("db.system", "postgresql")
                    db_span.set_attribute("db.operation", "INSERT")
                    db_span.set_attribute("db.table", "orders")
                    
                    order_id = await self._save_order(user_id, items, total, payment_result)
                    db_span.set_attribute("order.id", order_id)
                
                # Event สำหรับ milestone สำคัญ
                span.add_event(
                    "order_created",
                    attributes={
                        "order.id": order_id,
                        "order.status": "confirmed"
                    }
                )
                
                span.set_status(Status(StatusCode.OK))
                return {"order_id": order_id, "status": "confirmed"}
                
            except Exception as e:
                # บันทึก error
                span.record_exception(e)
                span.set_status(Status(StatusCode.ERROR, str(e)))
                raise
    
    async def _validate_order(self, items: list):
        """Validate order items"""
        await asyncio.sleep(0.005)
    
    async def _check_inventory(self, items: list) -> bool:
        """ตรวจสอบ inventory"""
        await asyncio.sleep(0.01)
        return True
    
    async def _process_payment(self, user_id: str, amount: float) -> dict:
        """ประมวลผลการชำระเงิน"""
        await asyncio.sleep(0.05)
        return {"transaction_id": f"TXN-{uuid.uuid4().hex[:8].upper()}"}
    
    async def _save_order(self, user_id: str, items: list, total: float, payment: dict) -> str:
        """บันทึก order ลง database"""
        await asyncio.sleep(0.02)
        return f"ORD-{uuid.uuid4().hex[:8].upper()}"


# Decorator-based tracing
def traced(span_name: str = None, attributes: dict = None):
    """Decorator สำหรับ tracing function"""
    def decorator(func: Callable):
        name = span_name or f"{func.__module__}.{func.__qualname__}"
        
        @functools.wraps(func)
        async def async_wrapper(*args, **kwargs):
            with tracer.start_as_current_span(name) as span:
                # เพิ่ม attributes ที่กำหนด
                for key, value in (attributes or {}).items():
                    span.set_attribute(key, value)
                
                try:
                    result = await func(*args, **kwargs)
                    span.set_status(Status(StatusCode.OK))
                    return result
                except Exception as e:
                    span.record_exception(e)
                    span.set_status(Status(StatusCode.ERROR, str(e)))
                    raise
        
        @functools.wraps(func)
        def sync_wrapper(*args, **kwargs):
            with tracer.start_as_current_span(name) as span:
                for key, value in (attributes or {}).items():
                    span.set_attribute(key, value)
                
                try:
                    result = func(*args, **kwargs)
                    span.set_status(Status(StatusCode.OK))
                    return result
                except Exception as e:
                    span.record_exception(e)
                    span.set_status(Status(StatusCode.ERROR, str(e)))
                    raise
        
        if asyncio.iscoroutinefunction(func):
            return async_wrapper
        return sync_wrapper
    
    return decorator


# ตัวอย่างการใช้ decorator
@traced("payment.process", attributes={"payment.provider": "stripe"})
async def process_stripe_payment(amount: float, token: str):
    """ประมวลผลการชำระเงินผ่าน Stripe"""
    await asyncio.sleep(0.1)
    return {"status": "success"}
```

---

## 10. Sentry Error Tracking

```python
# sentry_setup.py
# Error tracking ด้วย Sentry SDK

import sentry_sdk
from sentry_sdk.integrations.fastapi import FastApiIntegration
from sentry_sdk.integrations.sqlalchemy import SqlalchemyIntegration
from sentry_sdk.integrations.redis import RedisIntegration
from sentry_sdk.integrations.logging import LoggingIntegration
from sentry_sdk.integrations.celery import CeleryIntegration
import logging
from typing import Optional
from fastapi import FastAPI, Request
import traceback


def initialize_sentry(
    dsn: str,
    environment: str = "production",
    release: str = "1.0.0",
    sample_rate: float = 1.0,
    traces_sample_rate: float = 0.1,  # 10% ของ transactions
    profiles_sample_rate: float = 0.05,  # 5% profiling
):
    """ตั้งค่า Sentry SDK"""
    
    sentry_logging = LoggingIntegration(
        level=logging.WARNING,        # capture warnings ขึ้นไป
        event_level=logging.ERROR,    # ส่ง errors ไป Sentry
    )
    
    sentry_sdk.init(
        dsn=dsn,
        environment=environment,
        release=release,
        
        # Integrations
        integrations=[
            FastApiIntegration(
                transaction_style="url",  # ใช้ URL pattern แทน function name
            ),
            SqlalchemyIntegration(),
            RedisIntegration(),
            sentry_logging,
            CeleryIntegration(),
        ],
        
        # Sampling rates
        sample_rate=sample_rate,                  # error sampling
        traces_sample_rate=traces_sample_rate,    # performance tracing
        profiles_sample_rate=profiles_sample_rate, # profiling
        
        # Before-send hook สำหรับ filter/modify events
        before_send=before_send_filter,
        before_send_transaction=before_send_transaction_filter,
        
        # ข้อมูลเพิ่มเติม
        send_default_pii=False,  # ไม่ส่ง PII (Personal Identifiable Information)
        
        # ตัวเลือก performance
        enable_tracing=True,
        
        # Filter ข้อมูลที่ sensitive
        event_scrubber=sentry_sdk.scrubber.EventScrubber(
            denylist=sentry_sdk.scrubber.DEFAULT_DENYLIST + [
                "password", "token", "secret", "api_key",
                "credit_card", "cvv", "ssn"
            ]
        )
    )


def before_send_filter(event, hint):
    """กรอง events ก่อนส่งไป Sentry"""
    
    # ไม่ส่ง 404 errors (ปกติเกิดจาก bots)
    if "exc_info" in hint:
        exc_type, exc_value, _ = hint["exc_info"]
        if hasattr(exc_value, 'status_code') and exc_value.status_code == 404:
            return None  # ไม่ส่ง event
    
    # ลบ sensitive data จาก headers
    if "request" in event and "headers" in event["request"]:
        headers = event["request"]["headers"]
        sensitive_headers = ["authorization", "cookie", "x-api-key"]
        for header in sensitive_headers:
            if header in headers:
                headers[header] = "[FILTERED]"
    
    return event


def before_send_transaction_filter(event, hint):
    """กรอง transaction events"""
    
    # ไม่ track health check endpoints
    if event.get("transaction") in ["/health", "/metrics", "/favicon.ico"]:
        return None
    
    return event


# ===== Custom Error Capture =====
class ErrorTracker:
    """Helper class สำหรับ capture errors ไป Sentry"""
    
    @staticmethod
    def capture_exception(
        error: Exception,
        user_id: str = None,
        extra_context: dict = None,
        tags: dict = None,
        level: str = "error"
    ):
        """Capture exception ไป Sentry พร้อม context"""
        
        with sentry_sdk.push_scope() as scope:
            # ตั้งค่า user context
            if user_id:
                scope.set_user({"id": user_id})
            
            # เพิ่ม extra context
            if extra_context:
                for key, value in extra_context.items():
                    scope.set_extra(key, value)
            
            # เพิ่ม tags สำหรับ filtering
            if tags:
                for key, value in tags.items():
                    scope.set_tag(key, value)
            
            scope.set_level(level)
            sentry_sdk.capture_exception(error)
    
    @staticmethod
    def capture_message(
        message: str,
        level: str = "info",
        user_id: str = None,
        extra_context: dict = None
    ):
        """Capture custom message ไป Sentry"""
        
        with sentry_sdk.push_scope() as scope:
            if user_id:
                scope.set_user({"id": user_id})
            
            if extra_context:
                for key, value in extra_context.items():
                    scope.set_extra(key, value)
            
            scope.set_level(level)
            sentry_sdk.capture_message(message, level=level)
    
    @staticmethod
    def add_breadcrumb(
        category: str,
        message: str,
        data: dict = None,
        level: str = "info"
    ):
        """เพิ่ม breadcrumb เพื่อ track การทำงานก่อน error"""
        sentry_sdk.add_breadcrumb(
            category=category,
            message=message,
            data=data or {},
            level=level
        )


# ===== ตัวอย่างการใช้งาน =====
def example_usage():
    """ตัวอย่างการใช้ Sentry"""
    
    error_tracker = ErrorTracker()
    
    try:
        # Track breadcrumbs ก่อน operation
        error_tracker.add_breadcrumb(
            category="auth",
            message="User เริ่ม checkout process",
            data={"user_id": "USR-123", "cart_items": 3}
        )
        
        # ... ทำงาน ...
        raise ValueError("ไม่พบข้อมูล product")
        
    except ValueError as e:
        # Capture exception พร้อม context
        error_tracker.capture_exception(
            error=e,
            user_id="USR-123",
            extra_context={
                "order_id": "ORD-456",
                "cart_items": [{"id": "P001", "qty": 2}],
                "payment_method": "credit_card"
            },
            tags={
                "feature": "checkout",
                "severity": "high"
            }
        )
        raise


# FastAPI Sentry middleware
from fastapi import Request

async def sentry_context_middleware(request: Request, call_next):
    """Middleware สำหรับเพิ่ม request context ใน Sentry"""
    
    with sentry_sdk.configure_scope() as scope:
        # เพิ่ม request metadata
        scope.set_tag("http.method", request.method)
        scope.set_tag("http.url", str(request.url))
        
        # เพิ่ม user ถ้า login
        user_id = request.headers.get("X-User-ID")
        if user_id:
            scope.set_user({"id": user_id})
        
        response = await call_next(request)
        return response
```

---

## 11. Complete Observability Stack

```python
# complete_observability.py
# ตัวอย่าง Complete Observability Stack สำหรับ Production

from fastapi import FastAPI, Request, Response, HTTPException
from prometheus_client import Counter, Gauge, Histogram, generate_latest, CONTENT_TYPE_LATEST
import structlog
import sentry_sdk
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.resources import Resource, SERVICE_NAME
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
import time
import uuid
import asyncio
from contextlib import asynccontextmanager
from typing import Optional
import logging

# ===== 1. กำหนด Metrics =====
# Prometheus metrics
http_requests_total = Counter(
    'app_http_requests_total',
    'จำนวน HTTP requests ทั้งหมด',
    ['method', 'endpoint', 'status']
)

http_request_duration = Histogram(
    'app_http_request_duration_seconds',
    'ระยะเวลา HTTP request',
    ['method', 'endpoint'],
    buckets=[.005, .01, .025, .05, .075, .1, .25, .5, .75, 1.0, 2.5, 5.0]
)

active_requests = Gauge(
    'app_active_requests',
    'จำนวน requests ที่กำลังดำเนินการ'
)

business_orders_total = Counter(
    'app_business_orders_total',
    'จำนวน orders ทั้งหมด',
    ['status']
)

# ===== 2. ตั้งค่า Logging =====
def setup_logging():
    """ตั้งค่า structured logging"""
    structlog.configure(
        processors=[
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.stdlib.add_log_level,
            structlog.processors.CallsiteParameterAdder([
                structlog.processors.CallsiteParameter.FILENAME,
                structlog.processors.CallsiteParameter.LINENO,
            ]),
            structlog.processors.format_exc_info,
            structlog.processors.JSONRenderer()
        ],
        wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
        logger_factory=structlog.PrintLoggerFactory(),
        cache_logger_on_first_use=True,
    )

setup_logging()
logger = structlog.get_logger("app")

# ===== 3. ตั้งค่า Tracing =====
def setup_tracing():
    """ตั้งค่า OpenTelemetry"""
    resource = Resource.create({SERVICE_NAME: "ecommerce-api"})
    provider = TracerProvider(resource=resource)
    trace.set_tracer_provider(provider)
    return trace.get_tracer("ecommerce-api")

app_tracer = setup_tracing()

# ===== 4. ตั้งค่า Sentry =====
sentry_sdk.init(
    dsn="https://your-dsn@sentry.io/project-id",  # ใส่ DSN จริง
    environment="production",
    traces_sample_rate=0.1,
)

# ===== 5. สร้าง FastAPI App =====
@asynccontextmanager
async def lifespan(app: FastAPI):
    logger.info("application_started", service="ecommerce-api", version="1.0.0")
    yield
    logger.info("application_stopped")

app = FastAPI(title="Fully Observable API", lifespan=lifespan)

# Instrument FastAPI กับ OpenTelemetry
FastAPIInstrumentor.instrument_app(app)


# ===== 6. Observability Middleware =====
@app.middleware("http")
async def observability_middleware(request: Request, call_next):
    """Middleware รวม metrics + logging + tracing"""
    
    request_id = str(uuid.uuid4())
    start_time = time.time()
    
    # สร้าง log context
    log = logger.bind(
        request_id=request_id,
        method=request.method,
        path=request.url.path,
        client_ip=request.client.host if request.client else "unknown"
    )
    
    log.info("request_started")
    active_requests.inc()
    
    status_code = 500
    
    try:
        response = await call_next(request)
        status_code = response.status_code
        return response
        
    except Exception as e:
        log.error("request_failed", error=str(e), exc_info=True)
        sentry_sdk.capture_exception(e)
        raise
        
    finally:
        duration = time.time() - start_time
        active_requests.dec()
        
        # บันทึก Prometheus metrics
        endpoint = request.url.path
        http_requests_total.labels(
            method=request.method,
            endpoint=endpoint,
            status=str(status_code)
        ).inc()
        
        http_request_duration.labels(
            method=request.method,
            endpoint=endpoint
        ).observe(duration)
        
        # Log request completion
        log_func = log.info if status_code < 400 else log.warning if status_code < 500 else log.error
        log_func(
            "request_completed",
            status_code=status_code,
            duration_ms=round(duration * 1000, 2)
        )


# ===== 7. API Endpoints =====
@app.get("/metrics")
async def metrics():
    """Prometheus metrics endpoint"""
    return Response(
        content=generate_latest(),
        media_type=CONTENT_TYPE_LATEST
    )


@app.get("/health")
async def health():
    """Health check endpoint"""
    return {"status": "healthy", "timestamp": time.time()}


@app.get("/health/live")
async def liveness():
    """Kubernetes liveness probe"""
    return {"alive": True}


@app.get("/health/ready")
async def readiness():
    """Kubernetes readiness probe"""
    # ตรวจสอบว่าพร้อมรับ traffic หรือไม่
    db_ready = True   # ตรวจสอบ DB connection
    cache_ready = True  # ตรวจสอบ Cache connection
    
    if not (db_ready and cache_ready):
        raise HTTPException(
            status_code=503,
            detail="Service not ready"
        )
    
    return {"ready": True}


@app.post("/api/orders")
async def create_order(order_data: dict):
    """API สำหรับสร้าง order - พร้อม full observability"""
    
    order_log = logger.bind(
        operation="create_order",
        user_id=order_data.get("user_id")
    )
    
    with app_tracer.start_as_current_span("create_order") as span:
        span.set_attribute("order.user_id", order_data.get("user_id", ""))
        
        try:
            order_log.info("order_creation_started")
            
            # จำลองการทำงาน
            await asyncio.sleep(0.05)
            
            order_id = f"ORD-{uuid.uuid4().hex[:8].upper()}"
            
            # บันทึก business metric
            business_orders_total.labels(status="created").inc()
            
            order_log.info("order_created", order_id=order_id)
            span.set_attribute("order.id", order_id)
            
            return {"order_id": order_id, "status": "created"}
            
        except Exception as e:
            business_orders_total.labels(status="failed").inc()
            order_log.error("order_creation_failed", error=str(e))
            span.record_exception(e)
            raise


# docker-compose.yml สำหรับ observability stack
DOCKER_COMPOSE = """
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SENTRY_DSN=https://your-dsn@sentry.io/project
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
  
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
  
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
  
  otel-collector:
    image: otel/opentelemetry-collector:latest
    ports:
      - "4317:4317"
    volumes:
      - ./otel-config.yml:/etc/otel/config.yml
  
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"
      - "14268:14268"
  
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
"""
```

---

## 12. สรุป Part 102

✅ เข้าใจ Observability: Metrics, Logs, Traces สามเสาหลักของการ monitor ระบบ  
✅ ใช้ prometheus-client สร้าง Counter, Gauge, Histogram, Summary  
✅ Integrate Prometheus กับ FastAPI และ Django  
✅ เข้าใจ Grafana dashboard concepts และ alerting rules  
✅ ใช้ structlog สำหรับ structured logging  
✅ ตั้งค่า JSON logging format สำหรับ ELK Stack  
✅ Implement OpenTelemetry distributed tracing  
✅ ใช้ Sentry SDK สำหรับ error tracking และ performance monitoring  
✅ สร้าง Health check endpoints (basic, detailed, liveness, readiness)  
✅ รวมทุกอย่างเป็น Complete Observability Stack  

---

## ➡️ ถัดไป: Part 103 - Security Advanced

*Part 102/105 | Python Course - World-Class Level*
