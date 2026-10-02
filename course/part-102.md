# Part 102: Monitoring and Observability

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- ตั้งค่า Prometheus สำหรับ metrics collection
- สร้าง Grafana Dashboards สำหรับ visualization
- ใช้ Structured Logging ด้วย structlog
- Implement Distributed Tracing ด้วย OpenTelemetry
- ตั้งค่า Sentry สำหรับ error tracking
- สร้าง Alerting rules

---

## 1. Prometheus Metrics

```python
# monitoring/prometheus_setup.py
"""
Prometheus Metrics สำหรับ Python Web Applications

Types of Metrics:
1. Counter: ค่าที่เพิ่มขึ้นเท่านั้น (requests, errors)
2. Gauge: ค่าที่ขึ้นลงได้ (active connections, memory usage)
3. Histogram: distribution ของค่า (latency, request size)
4. Summary: คล้าย Histogram แต่คำนวณ quantiles ฝั่ง client
"""
from prometheus_client import (
    Counter, Gauge, Histogram, Summary,
    start_http_server, CollectorRegistry,
    multiprocess, generate_latest, CONTENT_TYPE_LATEST
)
from prometheus_client.core import GaugeMetricFamily, CounterMetricFamily
import time
import os
import psutil
import asyncio
from fastapi import FastAPI, Request, Response
from contextlib import asynccontextmanager
import threading


# ==================== Custom Metrics ====================

# HTTP Metrics
http_requests_total = Counter(
    "http_requests_total",
    "Total number of HTTP requests",
    ["method", "endpoint", "status_code", "service"]
)

http_request_duration_seconds = Histogram(
    "http_request_duration_seconds",
    "HTTP request latency in seconds",
    ["method", "endpoint", "service"],
    buckets=(0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0)
)

http_request_size_bytes = Histogram(
    "http_request_size_bytes",
    "HTTP request size in bytes",
    ["method", "endpoint"],
    buckets=(100, 1000, 10000, 100000, 1000000)
)

http_response_size_bytes = Histogram(
    "http_response_size_bytes",
    "HTTP response size in bytes",
    ["method", "endpoint"],
    buckets=(100, 1000, 10000, 100000, 1000000)
)

active_requests = Gauge(
    "http_active_requests",
    "Number of active HTTP requests",
    ["service"]
)

# Database Metrics
db_query_duration_seconds = Histogram(
    "db_query_duration_seconds",
    "Database query execution time",
    ["operation", "table"],
    buckets=(0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0, 5.0)
)

db_connection_pool_size = Gauge(
    "db_connection_pool_size",
    "Database connection pool size",
    ["pool_type"]  # active, idle, overflow
)

db_query_errors_total = Counter(
    "db_query_errors_total",
    "Total number of database query errors",
    ["error_type", "table"]
)

# Cache Metrics
cache_hits_total = Counter(
    "cache_hits_total",
    "Total number of cache hits",
    ["cache_type"]
)

cache_misses_total = Counter(
    "cache_misses_total",
    "Total number of cache misses",
    ["cache_type"]
)

cache_hit_ratio = Gauge(
    "cache_hit_ratio",
    "Current cache hit ratio (0-1)",
    ["cache_type"]
)

# Business Metrics
orders_created_total = Counter(
    "orders_created_total",
    "Total orders created",
    ["status"]  # success, failed
)

order_value_total = Counter(
    "order_value_total_baht",
    "Total order value in Thai Baht",
    []
)

payment_processing_duration = Histogram(
    "payment_processing_duration_seconds",
    "Payment processing time",
    ["payment_method"],
    buckets=(0.1, 0.5, 1.0, 2.0, 5.0, 10.0, 30.0)
)

# System Metrics
system_cpu_usage = Gauge("system_cpu_usage_percent", "System CPU usage")
system_memory_usage = Gauge("system_memory_usage_bytes", "System memory usage")
system_memory_total = Gauge("system_memory_total_bytes", "Total system memory")


# ==================== Metrics Middleware ====================

SERVICE_NAME = os.getenv("SERVICE_NAME", "api")


class PrometheusMiddleware:
    """FastAPI middleware สำหรับ collect HTTP metrics"""
    
    async def __call__(self, request: Request, call_next) -> Response:
        path = request.url.path
        method = request.method
        
        # Skip metrics endpoint เอง
        if path == "/metrics":
            return await call_next(request)
        
        # Normalize path (แทน path parameters ด้วย {param})
        normalized_path = self._normalize_path(path)
        
        # Track active requests
        active_requests.labels(service=SERVICE_NAME).inc()
        
        # Track request size
        content_length = request.headers.get("content-length", 0)
        if content_length:
            http_request_size_bytes.labels(
                method=method,
                endpoint=normalized_path
            ).observe(int(content_length))
        
        start_time = time.perf_counter()
        
        try:
            response = await call_next(request)
            status_code = str(response.status_code)
        except Exception as e:
            status_code = "500"
            raise
        finally:
            # Record duration
            duration = time.perf_counter() - start_time
            
            http_requests_total.labels(
                method=method,
                endpoint=normalized_path,
                status_code=status_code,
                service=SERVICE_NAME
            ).inc()
            
            http_request_duration_seconds.labels(
                method=method,
                endpoint=normalized_path,
                service=SERVICE_NAME
            ).observe(duration)
            
            active_requests.labels(service=SERVICE_NAME).dec()
        
        # Track response size
        response_content_length = response.headers.get("content-length", 0)
        if response_content_length:
            http_response_size_bytes.labels(
                method=method,
                endpoint=normalized_path
            ).observe(int(response_content_length))
        
        return response
    
    def _normalize_path(self, path: str) -> str:
        """แทน path parameters ด้วย {param}"""
        import re
        # แทนตัวเลขในเส้นทาง
        normalized = re.sub(r"/\d+", "/{id}", path)
        # แทน UUID
        normalized = re.sub(
            r"/[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}",
            "/{uuid}",
            normalized
        )
        return normalized


# ==================== System Metrics Collector ====================

def collect_system_metrics():
    """เก็บ system metrics ทุก 15 วินาที"""
    while True:
        # CPU
        system_cpu_usage.set(psutil.cpu_percent(interval=1))
        
        # Memory
        memory = psutil.virtual_memory()
        system_memory_usage.set(memory.used)
        system_memory_total.set(memory.total)
        
        time.sleep(15)


# เริ่ม background thread สำหรับ system metrics
system_metrics_thread = threading.Thread(
    target=collect_system_metrics,
    daemon=True
)


# ==================== FastAPI Integration ====================

@asynccontextmanager
async def lifespan(app: FastAPI):
    # เริ่ม system metrics collection
    system_metrics_thread.start()
    yield


app = FastAPI(lifespan=lifespan)
app.middleware("http")(PrometheusMiddleware())


@app.get("/metrics")
async def metrics():
    """Expose Prometheus metrics endpoint"""
    return Response(
        content=generate_latest(),
        media_type=CONTENT_TYPE_LATEST
    )


# ==================== Database Metrics Decorator ====================

import functools
from typing import Callable


def track_db_query(operation: str, table: str):
    """Decorator สำหรับ track database query metrics"""
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        async def async_wrapper(*args, **kwargs):
            start = time.perf_counter()
            try:
                result = await func(*args, **kwargs)
                duration = time.perf_counter() - start
                db_query_duration_seconds.labels(
                    operation=operation,
                    table=table
                ).observe(duration)
                return result
            except Exception as e:
                db_query_errors_total.labels(
                    error_type=type(e).__name__,
                    table=table
                ).inc()
                raise
        
        return async_wrapper
    return decorator


# ตัวอย่างการใช้
class OrderRepository:
    @track_db_query(operation="SELECT", table="orders")
    async def get_orders(self, user_id: int):
        # database query
        pass
    
    @track_db_query(operation="INSERT", table="orders")
    async def create_order(self, data: dict):
        # database insert
        # เพิ่ม business metrics
        orders_created_total.labels(status="success").inc()
        order_value_total.inc(data.get("total", 0))
        pass
```

---

## 2. Grafana Dashboard Configuration

```json
{
  "dashboard": {
    "title": "Python App - Production Dashboard",
    "panels": [
      {
        "title": "Request Rate (req/s)",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total{service=\"api\"}[5m])",
            "legendFormat": "{{method}} {{endpoint}} {{status_code}}"
          }
        ]
      },
      {
        "title": "Request Latency P95 (ms)",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) * 1000",
            "legendFormat": "P95 {{endpoint}}"
          }
        ]
      },
      {
        "title": "Error Rate (%)",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(http_requests_total{status_code=~\"5..\"}[5m]) / rate(http_requests_total[5m]) * 100",
            "legendFormat": "Error Rate"
          }
        ]
      }
    ]
  }
}
```

```python
# monitoring/grafana_manager.py
import requests
import json
from typing import Optional


class GrafanaManager:
    """จัดการ Grafana dashboards ผ่าน API"""
    
    def __init__(self, url: str, api_key: str):
        self.url = url.rstrip("/")
        self.headers = {
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json"
        }
    
    def create_dashboard(self, dashboard: dict, folder_id: int = 0) -> dict:
        """สร้าง dashboard"""
        payload = {
            "dashboard": {**dashboard, "id": None},
            "folderId": folder_id,
            "overwrite": True
        }
        
        response = requests.post(
            f"{self.url}/api/dashboards/db",
            json=payload,
            headers=self.headers
        )
        response.raise_for_status()
        return response.json()
    
    def create_alert_rule(
        self,
        name: str,
        expression: str,
        threshold: float,
        severity: str = "warning",
        notification_channels: list = None
    ) -> dict:
        """สร้าง alert rule"""
        rule = {
            "name": name,
            "type": "alerting",
            "expr": expression,
            "for": "5m",
            "labels": {"severity": severity},
            "annotations": {
                "summary": f"Alert: {name}",
                "description": f"Value exceeded threshold {threshold}"
            }
        }
        
        response = requests.post(
            f"{self.url}/api/ruler/grafana/api/v1/rules/default",
            json={"name": "default", "rules": [rule]},
            headers=self.headers
        )
        response.raise_for_status()
        return response.json()
    
    def create_python_app_dashboard(self) -> dict:
        """สร้าง dashboard สำเร็จรูปสำหรับ Python app"""
        dashboard = {
            "title": "Python App Overview",
            "tags": ["python", "api", "production"],
            "timezone": "Asia/Bangkok",
            "refresh": "30s",
            "panels": [
                # Request Rate
                {
                    "id": 1,
                    "title": "Requests Per Second",
                    "type": "timeseries",
                    "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0},
                    "targets": [{
                        "expr": "sum(rate(http_requests_total[5m])) by (service)",
                        "legendFormat": "{{service}}"
                    }]
                },
                # Error Rate
                {
                    "id": 2,
                    "title": "Error Rate (%)",
                    "type": "stat",
                    "gridPos": {"h": 4, "w": 6, "x": 12, "y": 0},
                    "targets": [{
                        "expr": "sum(rate(http_requests_total{status_code=~'5..'}[5m])) / sum(rate(http_requests_total[5m])) * 100",
                        "legendFormat": "Error %"
                    }],
                    "fieldConfig": {
                        "defaults": {
                            "thresholds": {
                                "steps": [
                                    {"color": "green", "value": 0},
                                    {"color": "yellow", "value": 1},
                                    {"color": "red", "value": 5}
                                ]
                            },
                            "unit": "percent"
                        }
                    }
                },
                # P99 Latency
                {
                    "id": 3,
                    "title": "Latency P50/P95/P99",
                    "type": "timeseries",
                    "gridPos": {"h": 8, "w": 12, "x": 0, "y": 8},
                    "targets": [
                        {
                            "expr": "histogram_quantile(0.5, rate(http_request_duration_seconds_bucket[5m])) * 1000",
                            "legendFormat": "P50"
                        },
                        {
                            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) * 1000",
                            "legendFormat": "P95"
                        },
                        {
                            "expr": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) * 1000",
                            "legendFormat": "P99"
                        }
                    ],
                    "fieldConfig": {"defaults": {"unit": "ms"}}
                },
                # Active Connections
                {
                    "id": 4,
                    "title": "Active Requests",
                    "type": "gauge",
                    "gridPos": {"h": 4, "w": 6, "x": 18, "y": 0},
                    "targets": [{
                        "expr": "sum(http_active_requests)",
                        "legendFormat": "Active"
                    }]
                },
                # CPU & Memory
                {
                    "id": 5,
                    "title": "System Resources",
                    "type": "timeseries",
                    "gridPos": {"h": 8, "w": 12, "x": 12, "y": 8},
                    "targets": [
                        {
                            "expr": "system_cpu_usage_percent",
                            "legendFormat": "CPU %"
                        },
                        {
                            "expr": "system_memory_usage_bytes / system_memory_total_bytes * 100",
                            "legendFormat": "Memory %"
                        }
                    ]
                }
            ]
        }
        
        return self.create_dashboard(dashboard)
```

---

## 3. Structured Logging

```python
# logging/structured_logger.py
"""
Structured Logging ด้วย structlog
ทำให้ logs searchable และ analyzable ง่าย
"""
import structlog
import logging
import sys
import json
import uuid
import time
from datetime import datetime
from typing import Any, Optional
from contextvars import ContextVar
import asyncio
from fastapi import Request, Response

# Context variable สำหรับ request tracking
request_id_var: ContextVar[str] = ContextVar("request_id", default="")
user_id_var: ContextVar[Optional[int]] = ContextVar("user_id", default=None)
trace_id_var: ContextVar[str] = ContextVar("trace_id", default="")


def add_request_context(logger, method, event_dict):
    """Processor: เพิ่ม request context ไปยังทุก log"""
    request_id = request_id_var.get("")
    user_id = user_id_var.get(None)
    trace_id = trace_id_var.get("")
    
    if request_id:
        event_dict["request_id"] = request_id
    if user_id:
        event_dict["user_id"] = user_id
    if trace_id:
        event_dict["trace_id"] = trace_id
    
    return event_dict


def add_service_info(logger, method, event_dict):
    """Processor: เพิ่ม service information"""
    import os
    event_dict["service"] = os.getenv("SERVICE_NAME", "api")
    event_dict["environment"] = os.getenv("ENVIRONMENT", "development")
    return event_dict


def censor_sensitive_data(logger, method, event_dict):
    """Processor: ซ่อนข้อมูล sensitive"""
    sensitive_keys = {
        "password", "token", "secret", "api_key",
        "credit_card", "cvv", "ssn"
    }
    
    def _censor(obj, path=""):
        if isinstance(obj, dict):
            return {
                k: "***CENSORED***" if k.lower() in sensitive_keys else _censor(v, f"{path}.{k}")
                for k, v in obj.items()
            }
        elif isinstance(obj, list):
            return [_censor(item, path) for item in obj]
        return obj
    
    return _censor(event_dict)


def configure_logging(log_level: str = "INFO", json_output: bool = True):
    """ตั้งค่า structured logging"""
    
    # Processors
    shared_processors = [
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.processors.TimeStamper(fmt="iso", utc=True),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        add_request_context,
        add_service_info,
        censor_sensitive_data,
    ]
    
    if json_output:
        shared_processors.append(structlog.processors.JSONRenderer())
    else:
        shared_processors.extend([
            structlog.dev.ConsoleRenderer(colors=True)
        ])
    
    structlog.configure(
        processors=shared_processors,
        wrapper_class=structlog.make_filtering_bound_logger(
            getattr(logging, log_level.upper())
        ),
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(),
        cache_logger_on_first_use=True,
    )
    
    # ตั้งค่า standard library logging
    logging.basicConfig(
        format="%(message)s",
        stream=sys.stdout,
        level=getattr(logging, log_level.upper())
    )


# Logger instance
logger = structlog.get_logger()


# Logging Middleware
class LoggingMiddleware:
    """FastAPI middleware สำหรับ request/response logging"""
    
    async def __call__(self, request: Request, call_next) -> Response:
        # สร้าง request ID
        request_id = request.headers.get("X-Request-ID") or str(uuid.uuid4())
        request_id_var.set(request_id)
        
        # ดึง user ID จาก JWT (ถ้ามี)
        auth = request.headers.get("Authorization", "")
        if auth.startswith("Bearer "):
            try:
                import jwt
                payload = jwt.decode(auth[7:], options={"verify_signature": False})
                user_id_var.set(payload.get("user_id"))
            except Exception:
                pass
        
        bound_logger = logger.bind(
            http_method=request.method,
            http_path=request.url.path,
            http_version=request.scope.get("http_version"),
            client_ip=request.client.host if request.client else None,
            user_agent=request.headers.get("user-agent", "")[:100]
        )
        
        bound_logger.info("request_started")
        start_time = time.perf_counter()
        
        try:
            response = await call_next(request)
            elapsed_ms = (time.perf_counter() - start_time) * 1000
            
            bound_logger.info(
                "request_completed",
                http_status=response.status_code,
                duration_ms=round(elapsed_ms, 2),
                response_size=response.headers.get("content-length")
            )
            
            response.headers["X-Request-ID"] = request_id
            return response
        
        except Exception as e:
            elapsed_ms = (time.perf_counter() - start_time) * 1000
            bound_logger.exception(
                "request_failed",
                error_type=type(e).__name__,
                error_message=str(e),
                duration_ms=round(elapsed_ms, 2)
            )
            raise


# Log correlation ข้าม services
class CorrelatedLogger:
    """Logger ที่ส่ง correlation ID ข้าม services"""
    
    def __init__(self, service_name: str):
        self.service = service_name
        self._logger = structlog.get_logger(service_name)
    
    def bind(self, **kwargs):
        return self._logger.bind(**kwargs)
    
    def info(self, event: str, **kwargs):
        self._logger.info(event, **kwargs)
    
    def error(self, event: str, **kwargs):
        self._logger.error(event, **kwargs)
    
    def warning(self, event: str, **kwargs):
        self._logger.warning(event, **kwargs)
    
    def with_request(self, request_id: str, trace_id: str = ""):
        """สร้าง logger ที่มี request context"""
        return self._logger.bind(
            request_id=request_id,
            trace_id=trace_id
        )


# ตัวอย่างการใช้งาน
service_logger = CorrelatedLogger("order-service")


async def create_order_with_logging(order_data: dict):
    """สร้าง order พร้อม structured logging"""
    
    log = service_logger.bind(
        order_user_id=order_data.get("user_id"),
        order_items_count=len(order_data.get("items", []))
    )
    
    log.info("order_creation_started")
    
    try:
        # Validate
        log.info("order_validation_started")
        # ... validation logic
        log.info("order_validation_completed")
        
        # Process
        log.info("order_processing_started")
        # ... processing logic
        
        order_id = "ORD-12345"
        log.info(
            "order_created_successfully",
            order_id=order_id,
            order_total=order_data.get("total", 0)
        )
        
        return {"order_id": order_id}
    
    except ValueError as e:
        log.warning("order_validation_failed", error=str(e))
        raise
    
    except Exception as e:
        log.error(
            "order_creation_failed",
            error_type=type(e).__name__,
            error=str(e)
        )
        raise
```

---

## 4. OpenTelemetry Distributed Tracing

```python
# tracing/opentelemetry_setup.py
"""
OpenTelemetry Distributed Tracing
ติดตาม requests ข้าม microservices
"""
from opentelemetry import trace, propagate
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import (
    BatchSpanProcessor,
    ConsoleSpanExporter
)
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.instrumentation.redis import RedisInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
from opentelemetry.propagators.composite import CompositePropagator
from opentelemetry.baggage.propagation import W3CBaggagePropagator
from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator
import os


def setup_tracing(
    service_name: str,
    service_version: str = "1.0.0",
    otlp_endpoint: str = None,
    debug: bool = False
) -> trace.Tracer:
    """ตั้งค่า OpenTelemetry tracing"""
    
    # Resource (service metadata)
    resource = Resource.create({
        "service.name": service_name,
        "service.version": service_version,
        "deployment.environment": os.getenv("ENVIRONMENT", "development")
    })
    
    # Tracer Provider
    provider = TracerProvider(resource=resource)
    
    # Exporters
    if debug:
        # Debug: output ไปยัง console
        provider.add_span_processor(
            BatchSpanProcessor(ConsoleSpanExporter())
        )
    
    # OTLP Exporter (Jaeger, Zipkin, Tempo, etc.)
    otlp_endpoint = otlp_endpoint or os.getenv(
        "OTEL_EXPORTER_OTLP_ENDPOINT",
        "http://localhost:4317"
    )
    
    otlp_exporter = OTLPSpanExporter(
        endpoint=otlp_endpoint,
        insecure=True
    )
    
    provider.add_span_processor(
        BatchSpanProcessor(
            otlp_exporter,
            max_queue_size=2048,
            max_export_batch_size=512,
            export_timeout_millis=30000
        )
    )
    
    # Set global tracer provider
    trace.set_tracer_provider(provider)
    
    # Set propagators (W3C TraceContext + Baggage)
    propagate.set_global_textmap(
        CompositePropagator([
            TraceContextTextMapPropagator(),
            W3CBaggagePropagator()
        ])
    )
    
    return trace.get_tracer(service_name, service_version)


def instrument_app(app):
    """Auto-instrument FastAPI application"""
    FastAPIInstrumentor.instrument_app(
        app,
        excluded_urls="/health,/metrics,/ready",
        tracer_provider=trace.get_tracer_provider()
    )
    
    SQLAlchemyInstrumentor().instrument(
        enable_commenter=True,
        commenter_options={}
    )
    
    RedisInstrumentor().instrument()
    
    HTTPXClientInstrumentor().instrument()


# Custom Span decorator
from opentelemetry import trace as otel_trace
from functools import wraps
import asyncio


def traced(span_name: str = None, attributes: dict = None):
    """Decorator สำหรับสร้าง custom spans"""
    def decorator(func):
        name = span_name or f"{func.__module__}.{func.__qualname__}"
        
        @wraps(func)
        async def async_wrapper(*args, **kwargs):
            tracer = otel_trace.get_tracer(__name__)
            
            with tracer.start_as_current_span(name) as span:
                # เพิ่ม attributes
                if attributes:
                    for k, v in attributes.items():
                        span.set_attribute(k, str(v))
                
                # เพิ่ม function arguments ที่ไม่ sensitive
                for i, arg in enumerate(args):
                    if isinstance(arg, (str, int, float, bool)):
                        span.set_attribute(f"arg.{i}", str(arg))
                
                try:
                    result = await func(*args, **kwargs)
                    span.set_status(otel_trace.Status(otel_trace.StatusCode.OK))
                    return result
                except Exception as e:
                    span.record_exception(e)
                    span.set_status(
                        otel_trace.Status(otel_trace.StatusCode.ERROR, str(e))
                    )
                    raise
        
        @wraps(func)
        def sync_wrapper(*args, **kwargs):
            tracer = otel_trace.get_tracer(__name__)
            
            with tracer.start_as_current_span(name) as span:
                if attributes:
                    for k, v in attributes.items():
                        span.set_attribute(k, str(v))
                
                try:
                    result = func(*args, **kwargs)
                    span.set_status(otel_trace.Status(otel_trace.StatusCode.OK))
                    return result
                except Exception as e:
                    span.record_exception(e)
                    span.set_status(
                        otel_trace.Status(otel_trace.StatusCode.ERROR, str(e))
                    )
                    raise
        
        if asyncio.iscoroutinefunction(func):
            return async_wrapper
        return sync_wrapper
    
    return decorator


# ตัวอย่างการใช้ custom spans
tracer = otel_trace.get_tracer("order-service")


@traced(span_name="order.create", attributes={"layer": "service"})
async def create_order_traced(user_id: int, items: list) -> dict:
    """สร้าง order พร้อม tracing"""
    
    span = otel_trace.get_current_span()
    span.set_attribute("user.id", user_id)
    span.set_attribute("order.items_count", len(items))
    
    with tracer.start_as_current_span("order.validate_inventory") as child_span:
        child_span.set_attribute("inventory.items_count", len(items))
        # ... validate inventory
        child_span.add_event("inventory_validated")
    
    with tracer.start_as_current_span("order.process_payment") as payment_span:
        payment_span.set_attribute("payment.amount", sum(i.get("price", 0) for i in items))
        # ... process payment
    
    order_id = f"ORD-{user_id}-{int(time.time())}"
    span.set_attribute("order.id", order_id)
    span.add_event("order_created", {"order_id": order_id})
    
    return {"order_id": order_id}
```

---

## 5. Sentry Error Tracking

```python
# error_tracking/sentry_setup.py
"""
Sentry สำหรับ Error Tracking และ Performance Monitoring
"""
import sentry_sdk
from sentry_sdk.integrations.fastapi import FastApiIntegration
from sentry_sdk.integrations.sqlalchemy import SqlalchemyIntegration
from sentry_sdk.integrations.redis import RedisIntegration
from sentry_sdk.integrations.httpx import HttpxIntegration
from sentry_sdk.integrations.logging import LoggingIntegration
import logging
import os


def setup_sentry(
    dsn: str = None,
    environment: str = "production",
    release: str = None,
    sample_rate: float = 1.0,
    traces_sample_rate: float = 0.1
):
    """ตั้งค่า Sentry"""
    
    dsn = dsn or os.getenv("SENTRY_DSN")
    
    if not dsn:
        print("Warning: SENTRY_DSN not configured")
        return
    
    sentry_sdk.init(
        dsn=dsn,
        environment=environment,
        release=release or os.getenv("APP_VERSION", "1.0.0"),
        sample_rate=sample_rate,
        traces_sample_rate=traces_sample_rate,
        profiles_sample_rate=0.1,  # Profiling
        
        integrations=[
            FastApiIntegration(transaction_style="endpoint"),
            SqlalchemyIntegration(),
            RedisIntegration(),
            HttpxIntegration(),
            LoggingIntegration(
                level=logging.INFO,
                event_level=logging.ERROR
            )
        ],
        
        # Filter sensitive data
        before_send=filter_sensitive_data,
        
        # Ignore certain errors
        ignore_errors=[
            KeyboardInterrupt,
        ],
        
        # Additional context
        attach_stacktrace=True,
        send_default_pii=False,  # ไม่ส่ง PII
        max_breadcrumbs=50
    )


def filter_sensitive_data(event, hint):
    """กรองข้อมูล sensitive ก่อนส่งไปยัง Sentry"""
    
    if "exception" in event:
        for exception in event["exception"].get("values", []):
            if "stacktrace" in exception:
                for frame in exception["stacktrace"].get("frames", []):
                    # ลบ sensitive variables จาก locals
                    if "vars" in frame:
                        for key in list(frame["vars"].keys()):
                            if any(s in key.lower() for s in [
                                "password", "token", "secret", "key", "auth"
                            ]):
                                frame["vars"][key] = "***"
    
    return event


# การใช้งาน Sentry
def capture_exception_with_context(
    error: Exception,
    user_id: int = None,
    extra_data: dict = None
):
    """Capture exception พร้อม context"""
    
    with sentry_sdk.push_scope() as scope:
        # เพิ่ม user context
        if user_id:
            scope.set_user({"id": user_id})
        
        # เพิ่ม extra data
        if extra_data:
            for key, value in extra_data.items():
                scope.set_extra(key, value)
        
        # เพิ่ม tags
        scope.set_tag("service", os.getenv("SERVICE_NAME", "api"))
        
        sentry_sdk.capture_exception(error)


# Sentry Performance Monitoring
def trace_function(name: str):
    """Decorator สำหรับ Sentry performance tracing"""
    def decorator(func):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            with sentry_sdk.start_span(op=name) as span:
                span.set_tag("function", func.__name__)
                return await func(*args, **kwargs)
        return wrapper
    return decorator
```

---

## 6. Alerting

```python
# alerting/alert_manager.py
"""
Alerting สำหรับ Python Applications
"""
from prometheus_client import Gauge
import asyncio
import httpx
import os
import json
from typing import List, Callable
from dataclasses import dataclass
from enum import Enum


class AlertSeverity(Enum):
    INFO = "info"
    WARNING = "warning"
    CRITICAL = "critical"


@dataclass
class Alert:
    name: str
    message: str
    severity: AlertSeverity
    labels: dict = None
    value: float = None


class AlertManager:
    """จัดการ alerts"""
    
    def __init__(self):
        self.channels: List[Callable] = []
    
    def add_channel(self, channel: Callable):
        """เพิ่ม notification channel"""
        self.channels.append(channel)
    
    async def fire(self, alert: Alert):
        """ส่ง alert ไปยังทุก channels"""
        tasks = [channel(alert) for channel in self.channels]
        await asyncio.gather(*tasks, return_exceptions=True)


class SlackNotifier:
    """ส่ง alerts ไปยัง Slack"""
    
    def __init__(self, webhook_url: str):
        self.webhook_url = webhook_url
    
    async def __call__(self, alert: Alert):
        color_map = {
            AlertSeverity.INFO: "#36a64f",
            AlertSeverity.WARNING: "#ffcc00",
            AlertSeverity.CRITICAL: "#ff0000"
        }
        
        payload = {
            "attachments": [{
                "color": color_map[alert.severity],
                "title": f"[{alert.severity.value.upper()}] {alert.name}",
                "text": alert.message,
                "fields": [
                    {"title": k, "value": str(v), "short": True}
                    for k, v in (alert.labels or {}).items()
                ],
                "footer": "Python App Monitor",
                "ts": int(asyncio.get_event_loop().time())
            }]
        }
        
        async with httpx.AsyncClient() as client:
            await client.post(self.webhook_url, json=payload)


# Prometheus Alerting Rules
PROMETHEUS_ALERT_RULES = """
# alerting_rules.yml
groups:
  - name: python_app
    interval: 30s
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          rate(http_requests_total{status_code=~"5.."}[5m]) /
          rate(http_requests_total[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High HTTP error rate"
          description: "Error rate is {{ $value | humanizePercentage }}"
      
      # High latency
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High API latency"
          description: "P99 latency is {{ $value | humanizeDuration }}"
      
      # Service down
      - alert: ServiceDown
        expr: up{job="python-app"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service is down"
          description: "{{ $labels.instance }} is down"
      
      # High CPU
      - alert: HighCPU
        expr: system_cpu_usage_percent > 80
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage"
          description: "CPU usage is {{ $value }}%"
"""
```

---

## 7. สรุป Part 102

✅ ตั้งค่า Prometheus metrics: Counter, Gauge, Histogram สำหรับ HTTP, DB, Business
✅ สร้าง Grafana Dashboard พร้อม alert rules
✅ ใช้ structlog สำหรับ structured logging พร้อม request context และ censoring
✅ ตั้งค่า OpenTelemetry distributed tracing พร้อม auto-instrumentation
✅ ใช้ Sentry สำหรับ error tracking พร้อม sensitive data filtering
✅ สร้าง Alerting system พร้อม Slack notifications

## ➡️ ถัดไป: Part 103 - Security Advanced

*Part 102/105 | Python Course - World-Class Level*
