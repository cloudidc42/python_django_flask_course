# Part 042: Celery Task Queue
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Task Queue และ Message Broker
- ติดตั้งและตั้งค่า Celery กับ Redis
- สร้าง Tasks ด้วย @app.task decorator
- ใช้ apply_async, delay สำหรับ async tasks
- ตั้งค่า Periodic Tasks ด้วย Celery Beat
- Monitor Tasks ด้วย Flower
- Best practices สำหรับ production

---

## 1. ทำไมต้องใช้ Task Queue?

```
ปัญหา:
- ส่งอีเมล (รอ 3-5 วินาที) → user รอ
- สร้าง PDF รายงาน (รอ 10-30 วินาที) → timeout
- Process รูปภาพ → ช้า
- ส่ง SMS หลายพัน → หมดเวลา

แก้ปัญหาด้วย Celery:
- HTTP request รับทันที
- Task ถูกส่งไป worker ทำ background
- Worker ทำงาน parallel หลายตัว
- Track status ของ task ได้
```

```
Architecture:
[Web App] → [Redis/RabbitMQ Broker] → [Celery Workers] → [Result Backend]
              (Message Queue)            (Background)       (Redis/DB)
```

```bash
# ติดตั้ง
pip install celery redis
pip install flower  # Monitoring UI
pip install django-celery-beat  # สำหรับ periodic tasks
pip install celery[redis]  # Redis support
```

---

## 2. Basic Celery Setup

```python
# === celery_app.py ===
from celery import Celery
import os

# สร้าง Celery instance
app = Celery(
    "myapp",                              # App name
    broker="redis://localhost:6379/0",    # Message broker
    backend="redis://localhost:6379/1",  # Result backend
    include=["myapp.tasks"]               # Import tasks
)

# Configuration
app.conf.update(
    # Task settings
    task_serializer="json",
    accept_content=["json"],
    result_serializer="json",
    timezone="Asia/Bangkok",
    enable_utc=True,
    
    # Result settings
    result_expires=3600,      # ผล task หมดอายุใน 1 ชั่วโมง
    
    # Worker settings
    worker_prefetch_multiplier=1,   # รับ task ทีละ 1 (ป้องกัน memory spike)
    task_acks_late=True,            # Acknowledge หลัง task สำเร็จ
    
    # Retry settings
    task_max_retries=3,
    task_default_retry_delay=60,   # รอ 60 วินาทีก่อน retry
    
    # Time limits
    task_soft_time_limit=300,       # Soft limit: 5 นาที
    task_time_limit=600,            # Hard limit: 10 นาที
)

if __name__ == "__main__":
    app.start()
```

```python
# === tasks.py ===
from celery_app import app
from celery import shared_task, Task
import time
import logging

logger = logging.getLogger(__name__)


# === Simple Task ===
@app.task
def add(x: int, y: int) -> int:
    """Task ง่ายๆ สำหรับทดสอบ"""
    return x + y


@app.task
def send_email(to: str, subject: str, body: str) -> dict:
    """ส่งอีเมล - ทำงาน background"""
    logger.info(f"Sending email to {to}")
    
    # Simulate email sending
    time.sleep(2)
    
    print(f"Email sent to {to}: {subject}")
    return {
        "status": "sent",
        "to": to,
        "subject": subject
    }


@app.task
def generate_report(user_id: int, report_type: str) -> str:
    """สร้างรายงาน - ใช้เวลานาน"""
    logger.info(f"Generating {report_type} report for user {user_id}")
    
    # Simulate report generation
    time.sleep(10)
    
    filename = f"report_{user_id}_{report_type}.pdf"
    return filename


@app.task
def process_image(image_path: str, operations: list) -> str:
    """Process รูปภาพ"""
    logger.info(f"Processing image: {image_path}")
    
    for op in operations:
        logger.info(f"  Applying operation: {op}")
        time.sleep(1)  # simulate processing
    
    return f"processed_{image_path}"
```

---

## 3. Task Signatures และ Options

```python
from celery_app import app
from tasks import add, send_email, generate_report
import time

# === delay() - วิธีง่ายที่สุด ===
def demo_delay():
    # delay() = apply_async() แบบง่าย
    result = add.delay(10, 20)
    
    print(f"Task ID: {result.id}")
    print(f"Task status: {result.status}")  # PENDING
    
    # รอผล (blocking)
    value = result.get(timeout=10)
    print(f"Result: {value}")  # 30
    
    # ตรวจสอบ status
    if result.successful():
        print("Task succeeded!")
    elif result.failed():
        print("Task failed!")


# === apply_async() - ควบคุมได้มากกว่า ===
def demo_apply_async():
    # ส่ง task ทันที
    result = add.apply_async(args=[10, 20])
    
    # ส่ง task พร้อม options
    result = send_email.apply_async(
        args=["user@example.com"],
        kwargs={"subject": "Hello", "body": "World"},
        
        # Scheduling options
        countdown=60,              # รอ 60 วินาทีก่อนทำงาน
        # eta=datetime(2024, 1, 1, 10, 0, 0),  # ทำงานตอนเวลานี้
        
        # Execution options
        priority=5,                # Priority (0=low, 9=high)
        queue="email_queue",       # ส่งไป queue เฉพาะ
        
        # Retry options
        max_retries=3,
        
        # Expiry
        expires=3600,              # Task หมดอายุใน 1 ชั่วโมง
        
        # Routing
        routing_key="email.high",
        exchange="notifications",
    )
    
    print(f"Task scheduled: {result.id}")
    return result


# === Chord - รันหลาย tasks แล้ว callback ===
from celery import chord, group, chain

def demo_chord():
    from tasks import process_image
    
    # รัน tasks พร้อมกัน แล้วรวมผล
    callback = add.s(0)  # callback task
    result = chord(
        [add.s(1, 2), add.s(3, 4), add.s(5, 6)],  # header tasks
        callback                                      # callback
    )()
    
    print(f"Chord result: {result.get()}")


# === Group - รัน tasks พร้อมกัน ===
def demo_group():
    from tasks import process_image
    
    job = group([
        add.s(1, 2),
        add.s(3, 4),
        add.s(5, 6),
    ])
    
    result = job.apply_async()
    results = result.get()
    print(f"Group results: {results}")  # [3, 7, 11]


# === Chain - รัน tasks ต่อเนื่อง ===
def demo_chain():
    # ผลของ task แรกจะเป็น input ของ task ถัดไป
    pipeline = chain(
        add.s(1, 2),   # result = 3
        add.s(10),     # result = 3 + 10 = 13
        add.s(100),    # result = 13 + 100 = 113
    )
    
    result = pipeline.apply_async()
    print(f"Chain result: {result.get()}")  # 113
```

---

## 4. Task Retry

```python
from celery_app import app
from celery.exceptions import MaxRetriesExceededError
import requests
import logging

logger = logging.getLogger(__name__)


@app.task(bind=True, max_retries=3, default_retry_delay=60)
def send_webhook(self, url: str, data: dict) -> dict:
    """
    ส่ง webhook พร้อม retry
    bind=True ทำให้เข้าถึง self (task instance) ได้
    """
    try:
        response = requests.post(url, json=data, timeout=10)
        response.raise_for_status()
        return {"status": "sent", "status_code": response.status_code}
        
    except requests.exceptions.ConnectionError as exc:
        logger.warning(f"Connection error, retrying... (attempt {self.request.retries + 1})")
        raise self.retry(
            exc=exc,
            countdown=2 ** self.request.retries * 60,  # Exponential backoff
        )
    except requests.exceptions.HTTPError as exc:
        if exc.response.status_code in (429, 500, 502, 503, 504):
            # Retry สำหรับ server errors
            logger.warning(f"Server error {exc.response.status_code}, retrying...")
            raise self.retry(exc=exc, countdown=120)
        else:
            # ไม่ retry สำหรับ 4xx errors อื่นๆ
            raise


@app.task(bind=True)
def process_payment(self, payment_id: str, amount: float) -> dict:
    """
    Process payment พร้อม sophisticated retry logic
    """
    logger.info(f"Processing payment {payment_id}: ${amount}")
    
    # อ่านข้อมูล retry
    retries = self.request.retries
    max_retries = self.max_retries
    
    try:
        # Simulate payment processing
        import random
        if random.random() < 0.3:  # 30% chance of failure
            raise Exception("Payment gateway unavailable")
        
        return {
            "payment_id": payment_id,
            "status": "completed",
            "amount": amount
        }
        
    except Exception as exc:
        if retries < max_retries:
            # Exponential backoff: 1m, 2m, 4m
            wait_time = 60 * (2 ** retries)
            logger.warning(
                f"Payment failed, retry {retries + 1}/{max_retries} "
                f"in {wait_time}s"
            )
            raise self.retry(exc=exc, countdown=wait_time)
        else:
            # หมด retries แล้ว - log และ alert
            logger.error(f"Payment {payment_id} failed after {max_retries} retries")
            
            # Send alert
            # notify_admin.delay(f"Payment {payment_id} failed!")
            
            raise  # Re-raise ให้ task เป็น FAILED


# === Task Callbacks ===
@app.task(
    bind=True,
    on_failure=None,  # จะตั้งใน decorator
    on_success=None,
    on_retry=None
)
def reliable_task(self, data: dict) -> dict:
    """Task พร้อม lifecycle callbacks"""
    return {"processed": data}


def on_task_failure(exc, task_id, args, kwargs, einfo):
    """Callback เมื่อ task fail"""
    logger.error(f"Task {task_id} failed: {exc}")
    # send_alert_email(f"Task failed: {exc}")


def on_task_success(retval, task_id, args, kwargs):
    """Callback เมื่อ task success"""
    logger.info(f"Task {task_id} succeeded: {retval}")


# ตั้ง callbacks ด้วย signals
from celery.signals import task_failure, task_success

@task_failure.connect
def handle_task_failure(sender=None, task_id=None, exception=None, **kwargs):
    logger.error(f"Task {task_id} failed with {exception}")

@task_success.connect
def handle_task_success(sender=None, result=None, **kwargs):
    logger.info(f"Task completed with result: {result}")
```

---

## 5. Periodic Tasks (Celery Beat)

```python
from celery_app import app
from celery.schedules import crontab
import datetime

# === celeryconfig.py - Periodic Tasks ===
app.conf.beat_schedule = {
    # ทุก 30 วินาที
    "check-system-health": {
        "task": "tasks.check_system_health",
        "schedule": 30.0,  # seconds
        "args": (),
    },
    
    # ทุกชั่วโมง
    "cleanup-expired-sessions": {
        "task": "tasks.cleanup_expired_sessions",
        "schedule": 3600.0,  # 1 hour
    },
    
    # ทุกวัน เวลา 2:00 AM (Bangkok timezone)
    "daily-backup": {
        "task": "tasks.backup_database",
        "schedule": crontab(hour=2, minute=0),
        "options": {"queue": "low_priority"},
    },
    
    # ทุกวันจันทร์ เวลา 9:00 AM
    "weekly-report": {
        "task": "tasks.generate_weekly_report",
        "schedule": crontab(
            day_of_week="monday",
            hour=9,
            minute=0
        ),
    },
    
    # วันที่ 1 ของทุกเดือน
    "monthly-billing": {
        "task": "tasks.process_monthly_billing",
        "schedule": crontab(
            day_of_month=1,
            hour=0,
            minute=30
        ),
    },
    
    # ทุก 5 นาที ในเวลาทำการ (9-17, จ-ศ)
    "monitor-orders": {
        "task": "tasks.monitor_pending_orders",
        "schedule": crontab(
            minute="*/5",
            hour="9-17",
            day_of_week="mon-fri"
        ),
    },
}


# === Periodic Task Functions ===
@app.task
def check_system_health() -> dict:
    """ตรวจสอบ health ของระบบ"""
    import psutil
    
    cpu = psutil.cpu_percent(interval=1)
    memory = psutil.virtual_memory().percent
    disk = psutil.disk_usage("/").percent
    
    health = {
        "timestamp": datetime.datetime.now().isoformat(),
        "cpu_percent": cpu,
        "memory_percent": memory,
        "disk_percent": disk,
        "status": "healthy" if all([cpu < 90, memory < 85, disk < 80]) else "warning"
    }
    
    if health["status"] == "warning":
        logger.warning(f"System health warning: {health}")
        # send_alert.delay("System health warning", str(health))
    
    return health


@app.task
def cleanup_expired_sessions() -> int:
    """ลบ sessions ที่หมดอายุ"""
    # ตัวอย่าง - ลบ sessions เก่ากว่า 24 ชั่วโมง
    from datetime import datetime, timedelta
    
    cutoff = datetime.now() - timedelta(hours=24)
    
    # deleted = Session.objects.filter(created_at__lt=cutoff).delete()
    deleted_count = 0  # Mock
    
    logger.info(f"Cleaned up {deleted_count} expired sessions")
    return deleted_count


@app.task
def generate_weekly_report() -> str:
    """สร้างรายงานประจำสัปดาห์"""
    logger.info("Generating weekly report...")
    
    # Generate report logic
    report_file = f"weekly_report_{datetime.date.today()}.pdf"
    
    # send_report_email.delay("admin@example.com", report_file)
    
    return report_file


@app.task
def backup_database() -> dict:
    """Backup ฐานข้อมูล"""
    logger.info("Starting database backup...")
    
    backup_file = f"backup_{datetime.datetime.now().strftime('%Y%m%d_%H%M%S')}.sql"
    
    return {
        "backup_file": backup_file,
        "size_mb": 0,  # Mock
        "timestamp": datetime.datetime.now().isoformat()
    }
```

---

## 6. Task Routing และ Priority Queues

```python
# === celery_config_advanced.py ===
from kombu import Queue, Exchange

# กำหนด Queues
app.conf.task_queues = [
    Queue("default",
          Exchange("default"),
          routing_key="default"),
    Queue("high_priority",
          Exchange("high_priority"),
          routing_key="high.#"),
    Queue("email",
          Exchange("email"),
          routing_key="email.#"),
    Queue("reports",
          Exchange("reports"),
          routing_key="report.#"),
    Queue("low_priority",
          Exchange("low"),
          routing_key="low.#"),
]

app.conf.task_default_queue = "default"
app.conf.task_default_exchange = "default"
app.conf.task_default_routing_key = "default"

# Task Routing
app.conf.task_routes = {
    "tasks.send_email": {"queue": "email"},
    "tasks.send_sms": {"queue": "email"},
    "tasks.generate_report": {"queue": "reports"},
    "tasks.cleanup_*": {"queue": "low_priority"},
    "tasks.process_payment": {"queue": "high_priority"},
}


# === Tasks ที่กำหนด queue ===
@app.task(queue="email")
def send_email_task(to: str, subject: str, body: str):
    pass


@app.task(queue="high_priority", priority=9)
def urgent_notification(user_id: int, message: str):
    pass


@app.task(queue="reports", time_limit=600)
def heavy_report_task(report_id: int):
    pass
```

---

## 7. สร้าง Custom Task Class

```python
from celery import Task
import logging
import time

logger = logging.getLogger(__name__)


class BaseTask(Task):
    """Base Task class พร้อม logging และ error handling"""
    
    abstract = True  # ไม่ register task นี้เอง
    
    def on_success(self, retval, task_id, args, kwargs):
        """เรียกเมื่อ task สำเร็จ"""
        logger.info(
            f"Task {self.name}[{task_id}] succeeded: {retval}"
        )
    
    def on_failure(self, exc, task_id, args, kwargs, einfo):
        """เรียกเมื่อ task fail"""
        logger.error(
            f"Task {self.name}[{task_id}] failed: {exc}\n{einfo}"
        )
        # อาจส่ง alert ที่นี่
    
    def on_retry(self, exc, task_id, args, kwargs, einfo):
        """เรียกเมื่อ task retry"""
        logger.warning(
            f"Task {self.name}[{task_id}] retrying: {exc}"
        )
    
    def __call__(self, *args, **kwargs):
        """Wrap task execution"""
        start_time = time.time()
        
        try:
            result = super().__call__(*args, **kwargs)
            elapsed = time.time() - start_time
            
            if elapsed > 30:  # Log ถ้าใช้เวลานานกว่า 30 วินาที
                logger.warning(
                    f"Task {self.name} took {elapsed:.2f}s (slow!)"
                )
            
            return result
        except Exception:
            raise


class DatabaseTask(BaseTask):
    """Task ที่ต้องการ DB connection"""
    
    abstract = True
    _db = None
    
    @property
    def db(self):
        """Lazy database connection"""
        if self._db is None:
            # เชื่อม database
            # self._db = create_engine(settings.DATABASE_URL)
            pass
        return self._db


# ใช้ custom base task
@app.task(base=BaseTask)
def monitored_task(data: dict) -> dict:
    """Task ที่ใช้ BaseTask class"""
    time.sleep(1)
    return {"processed": True, "data": data}


@app.task(base=DatabaseTask)
def db_intensive_task(user_id: int) -> list:
    """Task ที่ใช้ DatabaseTask"""
    # db = db_intensive_task.db  # Access DB
    return [{"user_id": user_id, "data": "example"}]
```

---

## 8. Flower Monitoring

```bash
# === การรัน Celery Worker ===

# รัน worker (1 worker)
celery -A celery_app worker --loglevel=info

# รัน worker หลายตัว (concurrency)
celery -A celery_app worker --concurrency=4 --loglevel=info

# รัน worker สำหรับ queue เฉพาะ
celery -A celery_app worker -Q email,notifications --loglevel=info

# รัน worker พร้อม pool settings
celery -A celery_app worker \
  --concurrency=4 \
  --pool=prefork \
  --max-tasks-per-child=1000 \
  --loglevel=info

# === รัน Celery Beat (Scheduler) ===
celery -A celery_app beat --loglevel=info

# รัน Beat พร้อม schedule file
celery -A celery_app beat \
  --scheduler django_celery_beat.schedulers:DatabaseScheduler \
  --loglevel=info

# === รัน Flower (Monitoring) ===
# pip install flower
celery -A celery_app flower \
  --port=5555 \
  --basic_auth=admin:password

# เปิด browser ไปที่ http://localhost:5555
```

```python
# === ตั้งค่า Flower ===
# flower_config.py

class FlowerConfig:
    broker = "redis://localhost:6379/0"
    port = 5555
    address = "0.0.0.0"
    url_prefix = "/flower"
    
    # Security
    basic_auth = ["admin:password"]
    
    # Features
    inspect_timeout = 5.0
    enable_events = True
    
    # Persistent data
    db = "flower.db"
    persistent = True
```

---

## 9. Integration กับ FastAPI

```python
# === main.py (FastAPI) ===
from fastapi import FastAPI, BackgroundTasks
from celery_app import app as celery_app
from tasks import send_email, generate_report, process_image
from celery.result import AsyncResult
from pydantic import BaseModel
from typing import Optional

api = FastAPI()


# === Request/Response Models ===
class EmailRequest(BaseModel):
    to: str
    subject: str
    body: str


class ReportRequest(BaseModel):
    user_id: int
    report_type: str


class TaskResponse(BaseModel):
    task_id: str
    status: str
    message: str


# === API Endpoints ===
@api.post("/send-email", response_model=TaskResponse)
async def api_send_email(request: EmailRequest):
    """ส่ง email background"""
    
    task = send_email.apply_async(
        args=[request.to, request.subject, request.body]
    )
    
    return TaskResponse(
        task_id=task.id,
        status="queued",
        message=f"Email queued for {request.to}"
    )


@api.post("/generate-report", response_model=TaskResponse)
async def api_generate_report(request: ReportRequest):
    """สร้างรายงาน background"""
    
    task = generate_report.apply_async(
        args=[request.user_id, request.report_type],
        queue="reports"
    )
    
    return TaskResponse(
        task_id=task.id,
        status="queued",
        message="Report generation started"
    )


@api.get("/task/{task_id}")
async def get_task_status(task_id: str):
    """ตรวจสอบ status ของ task"""
    
    result = AsyncResult(task_id, app=celery_app)
    
    response = {
        "task_id": task_id,
        "status": result.status,
        "ready": result.ready(),
        "successful": result.successful() if result.ready() else None,
    }
    
    if result.ready():
        if result.successful():
            response["result"] = result.get()
        else:
            response["error"] = str(result.result)
    
    return response


@api.get("/tasks")
async def list_active_tasks():
    """ดู tasks ที่กำลังทำงาน"""
    i = celery_app.control.inspect()
    
    return {
        "active": i.active(),      # กำลังทำงาน
        "scheduled": i.scheduled(), # กำหนดไว้ล่วงหน้า
        "reserved": i.reserved(),  # รอใน queue
    }


@api.post("/cancel/{task_id}")
async def cancel_task(task_id: str):
    """ยกเลิก task"""
    celery_app.control.revoke(task_id, terminate=True)
    return {"message": f"Task {task_id} cancelled"}


# Polling Example (client-side)
"""
async function waitForTask(taskId) {
  while (true) {
    const resp = await fetch(`/task/${taskId}`);
    const data = await resp.json();
    
    if (data.status === 'SUCCESS') {
      return data.result;
    } else if (data.status === 'FAILURE') {
      throw new Error(data.error);
    }
    
    await new Promise(r => setTimeout(r, 1000));  // Poll every 1s
  }
}
"""
```

---

## 10. Docker Compose Setup

```yaml
# docker-compose.yml
version: "3.8"

services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes
  
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      - REDIS_URL=redis://redis:6379/0
      - DATABASE_URL=postgresql://postgres:pass@db:5432/myapp
    depends_on:
      - redis
      - db
    command: uvicorn main:api --host 0.0.0.0 --port 8000
  
  celery_worker:
    build: .
    environment:
      - REDIS_URL=redis://redis:6379/0
      - DATABASE_URL=postgresql://postgres:pass@db:5432/myapp
    depends_on:
      - redis
    command: celery -A celery_app worker --loglevel=info --concurrency=4
  
  celery_beat:
    build: .
    environment:
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      - redis
    command: celery -A celery_app beat --loglevel=info
  
  flower:
    build: .
    ports:
      - "5555:5555"
    environment:
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      - redis
    command: celery -A celery_app flower --port=5555

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: pass
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  redis_data:
  postgres_data:
```

---

## 11. สรุป Part 042

✅ **Celery Setup** - ติดตั้งและตั้งค่ากับ Redis broker  
✅ **@app.task** - สร้าง task ด้วย decorator  
✅ **delay/apply_async** - ส่ง tasks แบบ async  
✅ **Task Groups** - group, chain, chord สำหรับ complex workflows  
✅ **Retry Logic** - Exponential backoff, max retries  
✅ **Periodic Tasks** - Celery Beat, crontab schedules  
✅ **Task Routing** - Priority queues, custom routing  
✅ **Flower** - Monitoring UI สำหรับ tasks  
✅ **FastAPI Integration** - REST API สำหรับ task management  
✅ **Docker Setup** - Production-ready deployment  

**Best Practices:**
- ใช้ task_acks_late=True เพื่อป้องกัน task หาย
- ตั้ง time limits ป้องกัน zombie tasks
- Monitor ด้วย Flower เสมอ
- ใช้ separate queues สำหรับ task types ต่างๆ

## ➡️ ถัดไป: Part 044 - CI/CD Pipeline
*Part 042/100+ | Python Course - Beginner to World-Class*
