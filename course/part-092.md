# Part 092: FastAPI Background Tasks
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้
- ใช้ BackgroundTasks สำหรับงานที่รันหลัง response
- สร้าง async background tasks
- ใช้ Celery กับ FastAPI สำหรับ distributed tasks
- ทำ scheduled tasks
- จัดการ task status

---

## 1. BackgroundTasks พื้นฐาน

BackgroundTasks ใช้สำหรับงานที่:
- ไม่จำเป็นต้องรอผลก่อนส่ง response
- ทำงานหลัง request เสร็จ
- เช่น: ส่งอีเมล, บันทึก log, อัปเดต statistics

```python
# background_tasks.py

from fastapi import FastAPI, BackgroundTasks
import time
import smtplib
from email.mime.text import MIMEText

app = FastAPI()


# ---- Background Functions ----

def send_welcome_email(email: str, username: str):
    """ส่งอีเมลต้อนรับ (จำลอง)"""
    time.sleep(2)  # จำลองเวลาส่งอีเมล
    print(f"✉️ ส่งอีเมลต้อนรับไปยัง {email} สำหรับ {username}")


def update_user_statistics(user_id: int):
    """อัปเดต statistics ของ user"""
    time.sleep(1)
    print(f"📊 อัปเดต statistics ของ user {user_id}")


def write_log(message: str):
    """บันทึก log ลงไฟล์"""
    with open("app.log", "a") as f:
        f.write(f"{time.ctime()}: {message}\n")
    print(f"📝 บันทึก log: {message}")


# ---- Routes ----

@app.post("/users/register")
async def register_user(
    username: str,
    email: str,
    background_tasks: BackgroundTasks  # inject BackgroundTasks
):
    """สมัครสมาชิก + ส่งอีเมลต้อนรับใน background"""
    
    # สร้าง user (รวดเร็ว)
    user_id = 1  # จำลอง
    
    # เพิ่มงานที่จะทำใน background
    # งานนี้จะทำงาน หลัง จากที่ส่ง response แล้ว
    background_tasks.add_task(send_welcome_email, email, username)
    background_tasks.add_task(update_user_statistics, user_id)
    background_tasks.add_task(write_log, f"New user registered: {username}")
    
    # ส่ง response ทันที ไม่รอ background tasks เสร็จ
    return {
        "message": f"สมัครสมาชิกสำเร็จ! เราจะส่งอีเมลยืนยันไปที่ {email}",
        "user_id": user_id
    }


@app.post("/orders/{order_id}/confirm")
async def confirm_order(
    order_id: int,
    background_tasks: BackgroundTasks
):
    """ยืนยัน order + ส่งอีเมล + อัปเดต stock ใน background"""
    
    # ยืนยัน order ใน database (จำลอง)
    print(f"✅ ยืนยัน order {order_id}")
    
    # Background tasks
    background_tasks.add_task(send_order_confirmation_email, order_id)
    background_tasks.add_task(update_inventory, order_id)
    background_tasks.add_task(notify_warehouse, order_id)
    
    return {"message": f"ยืนยัน order {order_id} แล้ว"}


def send_order_confirmation_email(order_id: int):
    time.sleep(1)
    print(f"✉️ ส่งอีเมลยืนยัน order {order_id}")


def update_inventory(order_id: int):
    time.sleep(0.5)
    print(f"📦 อัปเดต inventory สำหรับ order {order_id}")


def notify_warehouse(order_id: int):
    time.sleep(0.3)
    print(f"🏭 แจ้ง warehouse เกี่ยวกับ order {order_id}")
```

---

## 2. Async Background Tasks

```python
# async_background.py

import asyncio
import httpx
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()


# Async background functions
async def send_webhook(url: str, data: dict):
    """ส่ง webhook แบบ async"""
    async with httpx.AsyncClient() as client:
        try:
            response = await client.post(url, json=data, timeout=10.0)
            print(f"Webhook sent: {response.status_code}")
        except Exception as e:
            print(f"Webhook failed: {e}")


async def process_image(image_path: str, user_id: int):
    """ประมวลผลรูปภาพ async"""
    await asyncio.sleep(2)  # จำลองการประมวลผล
    print(f"Image processed: {image_path} for user {user_id}")
    # ส่ง notification ให้ user
    await send_webhook(
        "http://localhost:8001/notifications",
        {"user_id": user_id, "message": "รูปภาพของคุณพร้อมแล้ว"}
    )


@app.post("/images/upload")
async def upload_image(
    image_url: str,
    user_id: int,
    background_tasks: BackgroundTasks
):
    """Upload รูปภาพ + ประมวลผลใน background"""
    
    # บันทึก metadata ทันที
    saved_path = f"/uploads/{user_id}/image.jpg"
    
    # ประมวลผลใน background (async)
    background_tasks.add_task(process_image, saved_path, user_id)
    
    return {
        "message": "อัปโหลดสำเร็จ! กำลังประมวลผลรูปภาพ",
        "image_path": saved_path
    }
```

---

## 3. Celery กับ FastAPI

Celery เหมาะสำหรับ tasks ที่:
- ใช้เวลานาน
- ต้องการ retry เมื่อล้มเหลว
- ต้องการ distributed processing
- ต้องการ scheduling

### ติดตั้ง
```bash
pip install celery redis
# ต้องรัน Redis server ด้วย
```

### Celery Setup
```python
# celery_app.py

from celery import Celery
import os

# สร้าง Celery instance
celery_app = Celery(
    "worker",
    broker=os.environ.get("REDIS_URL", "redis://localhost:6379/0"),
    backend=os.environ.get("REDIS_URL", "redis://localhost:6379/0"),
    include=["tasks"]  # module ที่มี tasks
)

# Configuration
celery_app.conf.update(
    task_serializer="json",
    accept_content=["json"],
    result_serializer="json",
    timezone="Asia/Bangkok",
    enable_utc=True,
    # Retry settings
    task_acks_late=True,
    task_reject_on_worker_lost=True,
    # Timeout
    task_soft_time_limit=300,  # 5 นาที
    task_time_limit=600,       # 10 นาที (hard limit)
    # Result expiry
    result_expires=3600,       # 1 ชั่วโมง
)
```

### Tasks
```python
# tasks.py

from celery_app import celery_app
import time
from typing import Optional


@celery_app.task(
    bind=True,
    max_retries=3,
    default_retry_delay=60  # retry หลัง 60 วินาที
)
def send_email_task(self, to: str, subject: str, body: str):
    """Task ส่งอีเมล"""
    try:
        # ส่งอีเมลจริงๆ
        time.sleep(1)  # จำลอง
        print(f"Email sent to {to}: {subject}")
        return {"status": "sent", "to": to}
    except Exception as exc:
        # Retry เมื่อล้มเหลว
        raise self.retry(exc=exc)


@celery_app.task
def generate_report(user_id: int, report_type: str):
    """สร้าง report (ใช้เวลานาน)"""
    print(f"Generating {report_type} report for user {user_id}")
    time.sleep(10)  # จำลองการสร้าง report
    
    report_path = f"/reports/{user_id}/{report_type}.pdf"
    return {"status": "done", "path": report_path}


@celery_app.task
def process_payment(order_id: int, amount: float, payment_method: str):
    """ประมวลผลการชำระเงิน"""
    print(f"Processing payment: order {order_id}, amount {amount}")
    time.sleep(2)
    
    # จำลอง
    success = True
    return {"status": "success" if success else "failed", "order_id": order_id}


# Scheduled Task (Celery Beat)
@celery_app.on_after_configure.connect
def setup_periodic_tasks(sender, **kwargs):
    """ตั้ง scheduled tasks"""
    # รันทุกวันเวลา 02:00
    sender.add_periodic_task(
        crontab(hour=2, minute=0),
        cleanup_old_files.s(),
        name='cleanup-daily'
    )
    
    # รันทุก 5 นาที
    sender.add_periodic_task(
        300.0,
        update_statistics.s(),
        name='update-stats-every-5-minutes'
    )


@celery_app.task
def cleanup_old_files():
    """ลบไฟล์เก่า"""
    print("Cleaning up old files...")


@celery_app.task
def update_statistics():
    """อัปเดต statistics"""
    print("Updating statistics...")
```

### ใช้ Celery ใน FastAPI
```python
# main.py

from fastapi import FastAPI, BackgroundTasks
from tasks import send_email_task, generate_report, process_payment
from celery.result import AsyncResult

app = FastAPI()


@app.post("/orders/{order_id}/pay")
async def pay_order(order_id: int, amount: float, method: str):
    """ชำระเงิน — ส่ง task ไปให้ Celery"""
    
    # ส่ง task ไปทำงานใน Celery worker
    task = process_payment.delay(order_id, amount, method)
    
    return {
        "message": "กำลังดำเนินการชำระเงิน",
        "task_id": task.id,  # ใช้ตรวจสอบ status ภายหลัง
        "order_id": order_id
    }


@app.get("/tasks/{task_id}")
async def get_task_status(task_id: str):
    """ดู status ของ Celery task"""
    result = AsyncResult(task_id)
    
    return {
        "task_id": task_id,
        "status": result.status,
        # PENDING, STARTED, SUCCESS, FAILURE, RETRY
        "result": result.result if result.ready() else None,
        "ready": result.ready()
    }


@app.post("/reports/generate")
async def generate_user_report(user_id: int, report_type: str = "monthly"):
    """สร้าง report ใน background"""
    task = generate_report.delay(user_id, report_type)
    
    return {
        "message": "กำลังสร้าง report",
        "task_id": task.id,
        "check_status_at": f"/tasks/{task.id}"
    }


@app.post("/users/register")
async def register_with_email(username: str, email: str):
    """สมัครสมาชิก + ส่งอีเมลต้อนรับผ่าน Celery"""
    
    # สร้าง user ใน database (จำลอง)
    user_id = 999
    
    # ส่งอีเมลผ่าน Celery (ไม่ blocking)
    send_email_task.delay(
        to=email,
        subject="ยินดีต้อนรับ!",
        body=f"สวัสดีคุณ {username} ยินดีต้อนรับสู่ระบบของเรา"
    )
    
    return {"user_id": user_id, "message": "สมัครสำเร็จ! ตรวจอีเมลของคุณ"}
```

### รัน Celery Worker
```bash
# รัน Celery worker
celery -A celery_app worker --loglevel=info

# รัน Celery Beat (สำหรับ scheduled tasks)
celery -A celery_app beat --loglevel=info

# Monitor ด้วย Flower (web UI)
pip install flower
celery -A celery_app flower --port=5555
```

---

## 4. Background Task Status Tracking

```python
# task_tracker.py

from fastapi import FastAPI, BackgroundTasks
from pydantic import BaseModel
from typing import Optional, Dict
from datetime import datetime, timezone
import uuid
import asyncio

app = FastAPI()

# ใน production ควรใช้ Redis หรือ database
tasks_store: Dict[str, dict] = {}


class TaskStatus(BaseModel):
    task_id: str
    status: str  # pending, running, completed, failed
    progress: int = 0  # 0-100
    result: Optional[dict] = None
    error: Optional[str] = None
    created_at: datetime
    completed_at: Optional[datetime] = None


def create_task_entry(task_id: str) -> dict:
    """สร้าง task entry"""
    entry = {
        "task_id": task_id,
        "status": "pending",
        "progress": 0,
        "result": None,
        "error": None,
        "created_at": datetime.now(timezone.utc).isoformat(),
        "completed_at": None
    }
    tasks_store[task_id] = entry
    return entry


async def process_large_file(task_id: str, file_path: str):
    """จำลองการประมวลผลไฟล์ขนาดใหญ่"""
    try:
        tasks_store[task_id]["status"] = "running"
        
        for i in range(10):
            await asyncio.sleep(1)  # จำลองงาน
            tasks_store[task_id]["progress"] = (i + 1) * 10
            print(f"Task {task_id}: {(i+1)*10}%")
        
        tasks_store[task_id].update({
            "status": "completed",
            "progress": 100,
            "result": {"processed_rows": 1000, "file": file_path},
            "completed_at": datetime.now(timezone.utc).isoformat()
        })
    
    except Exception as e:
        tasks_store[task_id].update({
            "status": "failed",
            "error": str(e),
            "completed_at": datetime.now(timezone.utc).isoformat()
        })


@app.post("/process/file")
async def process_file(
    file_path: str,
    background_tasks: BackgroundTasks
):
    """เริ่มประมวลผลไฟล์"""
    task_id = str(uuid.uuid4())
    create_task_entry(task_id)
    
    background_tasks.add_task(process_large_file, task_id, file_path)
    
    return {
        "task_id": task_id,
        "message": "เริ่มประมวลผลแล้ว",
        "check_status": f"/tasks/{task_id}"
    }


@app.get("/tasks/{task_id}", response_model=TaskStatus)
async def get_task_status(task_id: str):
    """ดู status ของ task"""
    task = tasks_store.get(task_id)
    if not task:
        from fastapi import HTTPException
        raise HTTPException(404, f"ไม่พบ task {task_id}")
    return task
```

---

## 5. สรุป Part 092

✅ **BackgroundTasks** รัน tasks หลัง response โดยใช้ `add_task()`  
✅ **Async background tasks** รองรับ `async def` functions  
✅ **Celery** สำหรับ distributed tasks ที่ต้องการ retry, scheduling  
✅ **Task ID** ใช้ติดตาม status ของ long-running tasks  
✅ **Celery Beat** สำหรับ scheduled/periodic tasks  
✅ **Task status tracking** บันทึก progress และ result  

---

## ➡️ ถัดไป: Part 093 - FastAPI Middleware and CORS

*Part 092/100+ | Python Course - Beginner to World-Class*
