# Part 072: Django Celery Integration

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจแนวคิด Task Queue และ Celery
- ติดตั้งและตั้งค่า Celery กับ Django
- สร้างและรัน Tasks
- ตั้งค่า Periodic Tasks ด้วย Celery Beat
- Monitor ด้วย Flower

---

## 1. Task Queue คืออะไร?

Task Queue ใช้สำหรับงานที่:
- ใช้เวลานาน (ส่งอีเมล, resize รูปภาพ)
- ต้องทำใน background
- ต้องทำซ้ำตามตาราง

```
ไม่มี Celery:
User → POST /register → สร้าง User → ส่งอีเมล (5 วินาที) → Response

มี Celery:
User → POST /register → สร้าง User → Queue task → Response (0.1 วินาที)
                                         ↓
                              Celery Worker → ส่งอีเมล (background)
```

---

## 2. Architecture

```
Django App → Message Broker (Redis/RabbitMQ) → Celery Worker
                     ↑                              ↓
            Celery Beat (scheduler)          Task Result (Redis/DB)
```

---

## 3. การติดตั้ง

```bash
# ติดตั้ง Celery และ Redis
pip install celery redis django-celery-results django-celery-beat

# ติดตั้ง Redis (Ubuntu)
sudo apt install redis-server
sudo service redis start

# ตรวจสอบ Redis
redis-cli ping  # ควรได้ PONG
```

---

## 4. ตั้งค่า Celery กับ Django

```python
# myproject/celery.py
import os
from celery import Celery

# กำหนด Django settings
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myproject.settings')

# สร้าง Celery app
app = Celery('myproject')

# โหลด config จาก Django settings (prefix CELERY_)
app.config_from_object('django.conf:settings', namespace='CELERY')

# Auto-discover tasks จากทุก installed apps
app.autodiscover_tasks()


@app.task(bind=True)
def debug_task(self):
    """Task สำหรับ debug"""
    print(f'Request: {self.request!r}')
```

```python
# myproject/__init__.py
from .celery import app as celery_app

__all__ = ('celery_app',)
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'django_celery_results',  # เก็บผล tasks ใน Django DB
    'django_celery_beat',     # Periodic tasks scheduler
]

# Celery Configuration
CELERY_BROKER_URL = 'redis://localhost:6379/0'
CELERY_RESULT_BACKEND = 'django-db'  # หรือ 'redis://localhost:6379/1'

# Serialization
CELERY_ACCEPT_CONTENT = ['application/json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'

# Timezone
CELERY_TIMEZONE = 'Asia/Bangkok'
CELERY_ENABLE_UTC = True

# Task settings
CELERY_TASK_TRACK_STARTED = True  # track ว่า task เริ่มทำงานแล้ว
CELERY_TASK_TIME_LIMIT = 30 * 60  # timeout 30 นาที
CELERY_TASK_SOFT_TIME_LIMIT = 25 * 60  # soft timeout 25 นาที

# Retry settings
CELERY_TASK_MAX_RETRIES = 3

# Result expiration
CELERY_RESULT_EXPIRES = 3600  # 1 ชั่วโมง

# Worker settings
CELERY_WORKER_CONCURRENCY = 4  # จำนวน worker processes
CELERY_WORKER_MAX_TASKS_PER_CHILD = 1000  # restart worker หลัง 1000 tasks

# Beat schedule (periodic tasks)
from celery.schedules import crontab

CELERY_BEAT_SCHEDULE = {
    'cleanup-old-sessions': {
        'task': 'myapp.tasks.cleanup_old_sessions',
        'schedule': crontab(hour=2, minute=0),  # ทุกวันตี 2
    },
    'send-daily-report': {
        'task': 'myapp.tasks.send_daily_report',
        'schedule': crontab(hour=8, minute=0, day_of_week='mon-fri'),  # จ-ศ 8 โมง
    },
    'update-statistics': {
        'task': 'myapp.tasks.update_statistics',
        'schedule': 60.0,  # ทุก 60 วินาที
    },
}
```

```bash
# migrate สำหรับ celery results และ beat
python manage.py migrate
```

---

## 5. สร้าง Tasks

```python
# myapp/tasks.py
from celery import shared_task
from celery.utils.log import get_task_logger

logger = get_task_logger(__name__)

# Task ง่ายๆ
@shared_task
def add(x, y):
    """Task บวกเลข"""
    return x + y


# Task ส่งอีเมล
@shared_task(bind=True, max_retries=3)
def send_welcome_email(self, user_id):
    """ส่งอีเมลยินดีต้อนรับ"""
    try:
        from django.contrib.auth import get_user_model
        from django.core.mail import send_mail
        from django.conf import settings
        
        User = get_user_model()
        user = User.objects.get(pk=user_id)
        
        logger.info(f'ส่งอีเมลยินดีต้อนรับถึง {user.email}')
        
        send_mail(
            subject='ยินดีต้อนรับสู่ระบบ!',
            message=f'สวัสดี {user.get_full_name() or user.username}!\n\nขอบคุณที่สมัครสมาชิก',
            from_email=settings.DEFAULT_FROM_EMAIL,
            recipient_list=[user.email],
            fail_silently=False,
        )
        
        logger.info(f'ส่งอีเมลสำเร็จถึง {user.email}')
        return {'status': 'success', 'email': user.email}
    
    except User.DoesNotExist:
        logger.error(f'ไม่พบ user ID: {user_id}')
        raise
    
    except Exception as exc:
        logger.error(f'ส่งอีเมลล้มเหลว: {exc}')
        # Retry task - รอเพิ่มขึ้นเรื่อยๆ (exponential backoff)
        raise self.retry(exc=exc, countdown=60 * (2 ** self.request.retries))


# Task ประมวลผลรูปภาพ
@shared_task(bind=True, time_limit=300)  # timeout 5 นาที
def process_image(self, image_path, user_id):
    """Resize และ optimize รูปภาพ"""
    from PIL import Image
    import os
    
    self.update_state(
        state='PROGRESS',
        meta={'current': 0, 'total': 3, 'status': 'กำลังเปิดไฟล์...'}
    )
    
    try:
        # เปิดรูปภาพ
        img = Image.open(image_path)
        
        self.update_state(
            state='PROGRESS',
            meta={'current': 1, 'total': 3, 'status': 'กำลัง resize...'}
        )
        
        # Resize
        max_size = (1920, 1080)
        img.thumbnail(max_size, Image.LANCZOS)
        
        self.update_state(
            state='PROGRESS',
            meta={'current': 2, 'total': 3, 'status': 'กำลังบันทึก...'}
        )
        
        # บันทึก
        output_path = image_path.replace('.', '_processed.')
        img.save(output_path, optimize=True, quality=85)
        
        return {
            'status': 'success',
            'original': image_path,
            'processed': output_path,
            'size': os.path.getsize(output_path)
        }
    
    except Exception as exc:
        logger.error(f'Process image failed: {exc}')
        raise self.retry(exc=exc, countdown=30, max_retries=2)


# Task รายงานสถิติ
@shared_task
def generate_monthly_report(year, month):
    """สร้างรายงานประจำเดือน"""
    from django.db.models import Count, Sum
    from .models import Article, Comment
    
    data = {
        'year': year,
        'month': month,
        'articles_published': Article.objects.filter(
            created_at__year=year,
            created_at__month=month,
            status='published'
        ).count(),
        'total_views': Article.objects.filter(
            created_at__year=year,
            created_at__month=month
        ).aggregate(total=Sum('views_count'))['total'] or 0,
        'comments_count': Comment.objects.filter(
            created_at__year=year,
            created_at__month=month
        ).count(),
    }
    
    # ส่งรายงานทางอีเมล
    from django.core.mail import send_mail
    from django.conf import settings
    
    send_mail(
        subject=f'รายงานประจำเดือน {month}/{year}',
        message=str(data),
        from_email=settings.DEFAULT_FROM_EMAIL,
        recipient_list=[settings.ADMIN_EMAIL],
    )
    
    return data
```

---

## 6. เรียก Tasks จาก Views

```python
# views.py
from django.contrib.auth.decorators import login_required
from django.http import JsonResponse
from .tasks import send_welcome_email, process_image, generate_monthly_report

def register_view(request):
    """Register และส่งอีเมลใน background"""
    if request.method == 'POST':
        # สร้าง user
        user = create_user(request.POST)
        
        # ส่ง task ไป background (non-blocking)
        send_welcome_email.delay(user.id)
        
        return JsonResponse({'status': 'ok', 'message': 'สมัครสมาชิกสำเร็จ'})


@login_required
def upload_and_process_image(request):
    """Upload รูปและ process ใน background"""
    if request.FILES.get('image'):
        image = request.FILES['image']
        
        # บันทึกไฟล์
        path = save_image(image)
        
        # ส่ง task พร้อม args
        task = process_image.delay(path, request.user.id)
        
        return JsonResponse({
            'task_id': task.id,
            'status': 'processing'
        })


@login_required
def check_task_status(request, task_id):
    """ตรวจสอบสถานะ task"""
    from celery.result import AsyncResult
    
    result = AsyncResult(task_id)
    
    response = {
        'task_id': task_id,
        'status': result.status,  # PENDING, STARTED, PROGRESS, SUCCESS, FAILURE
    }
    
    if result.status == 'PROGRESS':
        response['progress'] = result.info  # ข้อมูล progress
    
    elif result.status == 'SUCCESS':
        response['result'] = result.result  # ผลลัพธ์
    
    elif result.status == 'FAILURE':
        response['error'] = str(result.result)
    
    return JsonResponse(response)


# เรียก task แบบต่างๆ
def demo_task_calls():
    # delay() - รันใน background ทันที
    result = send_welcome_email.delay(1)
    print(f'Task ID: {result.id}')
    
    # apply_async() - มี options มากกว่า
    result = send_welcome_email.apply_async(
        args=[1],
        countdown=60,       # รอ 60 วินาทีก่อนรัน
        eta=datetime(2024, 1, 1, 8, 0),  # รันตอนนี้
        expires=3600,       # task หมดอายุใน 1 ชั่วโมง
        queue='priority',   # ใช้ queue พิเศษ
        retry=True,
        retry_policy={'max_retries': 3}
    )
    
    # apply() - รันทันที (synchronous, ไม่ใช้ broker)
    result = send_welcome_email.apply(args=[1])
    print(result.result)
    
    # si() - signature (สำหรับ chain)
    from celery import chain
    task_chain = chain(
        send_welcome_email.si(1),
        generate_monthly_report.si(2024, 1)
    )
    task_chain.delay()
```

---

## 7. Periodic Tasks

### วิธีที่ 1: ใน settings.py

```python
# settings.py
from celery.schedules import crontab

CELERY_BEAT_SCHEDULE = {
    'task-name': {
        'task': 'myapp.tasks.task_function',
        'schedule': crontab(minute=0, hour='*/4'),  # ทุก 4 ชั่วโมง
    },
}
```

### วิธีที่ 2: django-celery-beat (แนะนำ - จัดการจาก Admin)

```python
# ใช้ django-celery-beat จัดการ periodic tasks ผ่าน Django Admin
# settings.py
CELERY_BEAT_SCHEDULER = 'django_celery_beat.schedulers:DatabaseScheduler'
```

```bash
# รัน celery beat
celery -A myproject beat -l info --scheduler django_celery_beat.schedulers:DatabaseScheduler
```

---

## 8. รัน Celery

```bash
# รัน Celery Worker
celery -A myproject worker --loglevel=info

# รัน Celery Worker หลาย processes
celery -A myproject worker --concurrency=4 --loglevel=info

# รัน Celery Beat (scheduler)
celery -A myproject beat --loglevel=info

# รัน Worker + Beat พร้อมกัน (development only)
celery -A myproject worker --beat --loglevel=info
```

---

## 9. Flower (Monitoring)

```bash
pip install flower

# รัน Flower
celery -A myproject flower --port=5555

# เปิด http://localhost:5555
```

Flower แสดง:
- Tasks ที่กำลังรัน/เสร็จ/ล้มเหลว
- Worker status
- Task statistics
- Task details/logs

---

## 10. Task Chains และ Groups

```python
from celery import chain, group, chord

# Chain: tasks ทำงานต่อกัน output ของ task แรกเป็น input ของถัดไป
result = chain(
    process_image.s('/path/to/img.jpg'),   # .s() = signature
    upload_to_cdn.s(),
    update_database.s()
).delay()

# Group: tasks ทำงานพร้อมกัน (parallel)
result = group(
    send_email.s(user_id)
    for user_id in user_ids
).delay()

# Chord: group แล้ว callback เมื่อทั้งหมดเสร็จ
result = chord(
    group(process_chunk.s(chunk) for chunk in data_chunks),
    combine_results.s()
).delay()
```

### Task Priority

```python
# กำหนด priority ให้ task
send_welcome_email.apply_async(
    args=[user_id],
    priority=9,    # 0-9 สูงกว่า = สำคัญกว่า (rabbitmq)
    queue='high_priority'
)

# settings.py - กำหนด queues
CELERY_TASK_ROUTES = {
    'myapp.tasks.send_email': {'queue': 'email'},
    'myapp.tasks.process_image': {'queue': 'heavy'},
    'myapp.tasks.generate_report': {'queue': 'reports'},
}

# รัน worker แยก queue
# celery -A myproject worker -Q email --concurrency=4
# celery -A myproject worker -Q heavy --concurrency=2
```

### Retry Strategy

```python
@shared_task(
    bind=True,
    autoretry_for=(Exception,),          # retry ทุก exception
    retry_kwargs={'max_retries': 5},      # retry สูงสุด 5 ครั้ง
    retry_backoff=True,                   # exponential backoff
    retry_backoff_max=700,               # สูงสุด 700 วินาที
    retry_jitter=True,                    # เพิ่ม random เพื่อกระจาย load
)
def send_notification(self, user_id, message):
    """ส่ง notification พร้อม auto-retry"""
    from .models import Notification
    from .push import send_push_notification
    
    try:
        user = User.objects.get(pk=user_id)
        send_push_notification(user.device_token, message)
        
        Notification.objects.create(
            user=user,
            message=message,
            status='sent'
        )
    except User.DoesNotExist:
        # ไม่ retry ถ้า user ไม่มี
        raise Exception(f'User {user_id} not found')
```

---

## 11. สรุป Part 072

✅ **Celery** เป็น task queue สำหรับรัน background jobs
✅ **Message Broker** (Redis/RabbitMQ) เป็นตัวกลางส่ง tasks
✅ **@shared_task** สร้าง task ที่ใช้ได้ทุก app
✅ **task.delay()** ส่ง task ไป background
✅ **task.apply_async()** ส่ง task พร้อม options เช่น countdown, eta, expires
✅ **Celery Beat** รัน periodic tasks ตาม schedule
✅ **django-celery-beat** จัดการ periodic tasks ผ่าน Django Admin
✅ **Flower** monitor tasks และ workers
✅ **chain/group/chord** รวม tasks ทำงานต่อกันหรือพร้อมกัน
✅ **retry** จัดการ task ที่ล้มเหลวด้วย exponential backoff

## ➡️ ถัดไป: Part 073 - Django Caching

*Part 072/100+ | Python Course - Beginner to World-Class*
