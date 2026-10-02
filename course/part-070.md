# Part 070: Django Signals

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจแนวคิด Signals ใน Django
- ใช้ pre_save/post_save signals
- ใช้ pre_delete/post_delete signals
- ใช้ m2m_changed signal
- สร้าง Custom signals
- เขียน signal receivers
- ตัวอย่างการใช้งานจริง

---

## 1. Signals คืออะไร?

Django Signals เป็น observer pattern ที่ช่วยให้ส่วนต่างๆ ของโปรแกรมสื่อสารกันได้ โดยไม่ต้องพึ่งพากัน (loose coupling)

```
ก่อนใช้ Signals:
Article.save()  →  update_search_index()  # ต้องเรียกเอง
                →  send_notification()
                →  clear_cache()

หลังใช้ Signals:
Article.save()  →  post_save signal  →  update_search_index (receiver)
                                    →  send_notification (receiver)
                                    →  clear_cache (receiver)
```

**เมื่อไหร่ควรใช้ Signals:**
- เมื่อต้องทำงานหลายอย่างเมื่อ event เกิดขึ้น
- เมื่อ logic อยู่ใน app ที่แยกกัน
- เมื่อต้องการ decoupling

**เมื่อไหร่ไม่ควรใช้ Signals:**
- เมื่อ logic ง่ายๆ อยู่ใน app เดียวกัน (ใช้ override save() แทน)
- Signals ทำให้ code ยากต่อการ debug

---

## 2. Built-in Signals

```python
# Django มี signals หลายอย่าง:

from django.db.models.signals import (
    pre_init,      # ก่อน __init__
    post_init,     # หลัง __init__
    pre_save,      # ก่อน save()
    post_save,     # หลัง save()
    pre_delete,    # ก่อน delete()
    post_delete,   # หลัง delete()
    m2m_changed,   # เมื่อ ManyToManyField เปลี่ยน
    pre_migrate,   # ก่อน migrate
    post_migrate,  # หลัง migrate
)

from django.core.signals import (
    request_started,   # เมื่อ HTTP request เริ่ม
    request_finished,  # เมื่อ HTTP request เสร็จ
    got_request_exception,  # เมื่อเกิด exception ใน request
)

from django.contrib.auth.signals import (
    user_logged_in,
    user_logged_out,
    user_login_failed,
)
```

---

## 3. pre_save / post_save

```python
# models.py หรือ signals.py
from django.db.models.signals import pre_save, post_save
from django.dispatch import receiver
from django.contrib.auth import get_user_model
from .models import Article, UserProfile

User = get_user_model()

# วิธีที่ 1: ใช้ @receiver decorator (แนะนำ)
@receiver(post_save, sender=Article)
def article_post_save(sender, instance, created, **kwargs):
    """
    sender:   Model class ที่ส่ง signal (Article)
    instance: object ที่เพิ่งถูก save
    created:  True ถ้าเพิ่งสร้างใหม่, False ถ้า update
    raw:      True ถ้า save จาก fixtures
    using:    database alias
    """
    if created:
        # ส่ง notification เมื่อสร้างบทความใหม่
        print(f'สร้างบทความใหม่: {instance.title}')
        # send_new_article_notification(instance)
    else:
        # Clear cache เมื่อแก้ไขบทความ
        print(f'อัปเดตบทความ: {instance.title}')
        # cache.delete(f'article_{instance.pk}')


@receiver(pre_save, sender=Article)
def article_pre_save(sender, instance, **kwargs):
    """ทำอะไรก่อน save"""
    # Auto-generate slug ถ้ายังไม่มี
    if not instance.slug:
        from django.utils.text import slugify
        import uuid
        instance.slug = slugify(instance.title)
        if not instance.slug:
            instance.slug = str(uuid.uuid4())[:8]
    
    # Auto-generate excerpt ถ้ายังไม่มี
    if not instance.excerpt and instance.content:
        instance.excerpt = instance.content[:200]


# วิธีที่ 2: connect() method
def on_user_save(sender, instance, created, **kwargs):
    """สร้าง UserProfile เมื่อสร้าง User"""
    if created:
        UserProfile.objects.get_or_create(user=instance)

# เชื่อม signal
post_save.connect(on_user_save, sender=User)
```

### ตัวอย่างจริง: สร้าง Profile อัตโนมัติ

```python
# accounts/signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.conf import settings
from .models import UserProfile

@receiver(post_save, sender=settings.AUTH_USER_MODEL)
def create_user_profile(sender, instance, created, **kwargs):
    """สร้าง Profile เมื่อสร้าง User ใหม่"""
    if created:
        UserProfile.objects.create(user=instance)


@receiver(post_save, sender=settings.AUTH_USER_MODEL)
def save_user_profile(sender, instance, **kwargs):
    """บันทึก Profile เมื่อ User ถูก save"""
    try:
        instance.profile.save()
    except UserProfile.DoesNotExist:
        # กรณี Profile ยังไม่ได้สร้าง
        UserProfile.objects.create(user=instance)
```

---

## 4. pre_delete / post_delete

```python
from django.db.models.signals import pre_delete, post_delete
from django.dispatch import receiver
import os

@receiver(pre_delete, sender=Article)
def article_pre_delete(sender, instance, **kwargs):
    """ก่อนลบบทความ"""
    # บันทึก log
    print(f'กำลังลบบทความ: {instance.title} (ID: {instance.pk})')
    
    # เก็บ ID ไว้ใช้ใน post_delete
    # ไม่สามารถหยุดการลบได้ใน signal (ต้องใช้ ModelAdmin.delete_model)


@receiver(post_delete, sender=Article)
def article_post_delete(sender, instance, **kwargs):
    """หลังลบบทความ"""
    # ลบไฟล์รูปภาพ
    if instance.thumbnail:
        if os.path.isfile(instance.thumbnail.path):
            os.remove(instance.thumbnail.path)
    
    # Clear cache
    # cache.delete(f'article_{instance.pk}')
    
    # อัปเดต search index
    # search_index.delete_document(instance.pk)
    
    print(f'ลบบทความแล้ว: {instance.title}')


@receiver(post_delete, sender='auth.User')
def user_post_delete(sender, instance, **kwargs):
    """หลังลบ user - ลบข้อมูลที่เกี่ยวข้อง"""
    # ลบ avatar
    if hasattr(instance, 'profile') and instance.profile.avatar:
        if os.path.isfile(instance.profile.avatar.path):
            os.remove(instance.profile.avatar.path)
```

---

## 5. m2m_changed Signal

```python
from django.db.models.signals import m2m_changed
from django.dispatch import receiver

@receiver(m2m_changed, sender=Article.tags.through)
def article_tags_changed(sender, instance, action, pk_set, **kwargs):
    """
    action: 'pre_add', 'post_add', 'pre_remove', 'post_remove', 'pre_clear', 'post_clear'
    pk_set: set ของ PKs ที่เพิ่ม/ลบ (None ใน pre_clear/post_clear)
    instance: Article instance
    """
    
    if action == 'post_add':
        # เพิ่ม tags ใหม่
        from .models import Tag
        new_tags = Tag.objects.filter(pk__in=pk_set)
        print(f'เพิ่ม tags: {[t.name for t in new_tags]} ให้บทความ: {instance.title}')
    
    elif action == 'post_remove':
        # ลบ tags ออก
        print(f'ลบ tag IDs: {pk_set} จากบทความ: {instance.title}')
    
    elif action == 'post_clear':
        # ล้าง tags ทั้งหมด
        print(f'ล้าง tags ทั้งหมดของบทความ: {instance.title}')
    
    # อัปเดต tag count
    if action in ['post_add', 'post_remove', 'post_clear']:
        # update_article_search_index(instance)
        pass
```

---

## 6. Auth Signals

```python
from django.contrib.auth.signals import user_logged_in, user_logged_out, user_login_failed
from django.dispatch import receiver

@receiver(user_logged_in)
def user_logged_in_handler(sender, request, user, **kwargs):
    """เมื่อ user login สำเร็จ"""
    # บันทึก IP และเวลา
    ip = request.META.get('REMOTE_ADDR')
    print(f'{user.username} login จาก {ip}')
    
    # อัปเดต last_login IP
    from .models import LoginHistory
    LoginHistory.objects.create(
        user=user,
        ip_address=ip,
        action='login'
    )


@receiver(user_logged_out)
def user_logged_out_handler(sender, request, user, **kwargs):
    """เมื่อ user logout"""
    if user:
        print(f'{user.username} logout')


@receiver(user_login_failed)
def user_login_failed_handler(sender, credentials, request, **kwargs):
    """เมื่อ login ล้มเหลว"""
    ip = request.META.get('REMOTE_ADDR')
    username = credentials.get('username', '')
    print(f'Login failed: username={username}, IP={ip}')
    
    # บันทึก failed attempt
    # check_brute_force(ip, username)
```

---

## 7. Custom Signals

```python
# signals.py
from django.dispatch import Signal

# สร้าง custom signal
article_published = Signal()  # ส่ง signal เมื่อบทความเผยแพร่
article_viewed = Signal()      # ส่ง signal เมื่อมีคนดูบทความ
order_completed = Signal()     # ส่ง signal เมื่อ order สำเร็จ

# providing_args (deprecated แต่ใช้บอก documentation):
# article_published = Signal()  # args: article, publisher
```

```python
# ส่ง signal จาก view หรือ model
from .signals import article_published

class ArticleViewSet(viewsets.ModelViewSet):
    
    @action(detail=True, methods=['post'])
    def publish(self, request, pk=None):
        article = self.get_object()
        article.status = 'published'
        article.save()
        
        # ส่ง signal พร้อม data
        article_published.send(
            sender=article.__class__,
            article=article,
            publisher=request.user
        )
        
        return Response({'status': 'published'})
```

```python
# รับ signal
from django.dispatch import receiver
from .signals import article_published

@receiver(article_published)
def on_article_published(sender, article, publisher, **kwargs):
    """รับ signal เมื่อบทความเผยแพร่"""
    
    # ส่งอีเมลถึง subscribers
    send_article_notification(article)
    
    # อัปเดต search index
    update_search_index(article)
    
    # บันทึก activity
    print(f'{publisher.username} เผยแพร่บทความ: {article.title}')
```

---

## 8. การลงทะเบียน Signals

```python
# accounts/apps.py
from django.apps import AppConfig

class AccountsConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'accounts'
    verbose_name = 'บัญชีผู้ใช้'
    
    def ready(self):
        """โหลด signals เมื่อ app พร้อม"""
        import accounts.signals  # noqa - import เพื่อ register signals


# settings.py
INSTALLED_APPS = [
    'accounts.apps.AccountsConfig',  # ใช้ AppConfig แทน 'accounts'
]
```

```python
# accounts/signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.conf import settings
from .models import UserProfile

@receiver(post_save, sender=settings.AUTH_USER_MODEL)
def create_user_profile(sender, instance, created, **kwargs):
    if created:
        UserProfile.objects.create(user=instance)
```

---

## 9. ตัวอย่างการใช้งานจริง

### Cache Invalidation

```python
@receiver(post_save, sender=Article)
@receiver(post_delete, sender=Article)
def invalidate_article_cache(sender, instance, **kwargs):
    """Clear cache เมื่อ article เปลี่ยนแปลง"""
    from django.core.cache import cache
    
    # Clear cache สำหรับบทความนี้
    cache.delete(f'article_{instance.pk}')
    cache.delete(f'article_slug_{instance.slug}')
    
    # Clear cache list
    cache.delete('article_list')
    cache.delete(f'category_{instance.category_id}_articles')
```

### Elasticsearch Integration

```python
@receiver(post_save, sender=Article)
def update_search_index(sender, instance, **kwargs):
    """อัปเดต Elasticsearch index"""
    if instance.status == 'published':
        from .tasks import index_article_task
        # ทำใน background ด้วย Celery
        index_article_task.delay(instance.pk)

@receiver(post_delete, sender=Article)
def remove_from_search_index(sender, instance, **kwargs):
    """ลบจาก Elasticsearch"""
    from .tasks import delete_from_index_task
    delete_from_index_task.delay(instance.pk)
```

### Audit Log

```python
from django.db.models.signals import post_save, post_delete
from django.dispatch import receiver

@receiver(post_save)
def create_audit_log_on_save(sender, instance, created, **kwargs):
    """บันทึก audit log สำหรับ models ที่กำหนด"""
    from django.contrib.contenttypes.models import ContentType
    
    # เฉพาะ models ที่ต้องการ track
    tracked_models = ['Article', 'Category', 'User']
    if sender.__name__ not in tracked_models:
        return
    
    action = 'created' if created else 'updated'
    
    # บันทึก log
    # AuditLog.objects.create(
    #     content_type=ContentType.objects.get_for_model(sender),
    #     object_id=instance.pk,
    #     action=action,
    # )
```

---

## 10. Disconnect Signals และ ป้องกัน Infinite Loop

```python
# ระวัง Infinite Loop: save() -> signal -> save() -> signal -> ...

@receiver(post_save, sender=Article)
def update_article_stats(sender, instance, **kwargs):
    """อัปเดตสถิติ - ระวัง infinite loop!"""
    
    # BAD: ทำให้ infinite loop
    # instance.save()  # จะ trigger signal อีกครั้ง!
    
    # GOOD: ใช้ update() แทน save()
    Article.objects.filter(pk=instance.pk).update(
        word_count=len(instance.content.split())
    )
    
    # หรือ disconnect signal ชั่วคราว
    # post_save.disconnect(update_article_stats, sender=Article)
    # instance.save()
    # post_save.connect(update_article_stats, sender=Article)


# ป้องกัน signal ใน test
from unittest.mock import patch

# ใน test
with patch('myapp.signals.send_email_on_publish.delay'):
    article.status = 'published'
    article.save()
```

### Disconnect Signal ชั่วคราว

```python
from contextlib import contextmanager
from django.db.models.signals import post_save

@contextmanager
def mute_signals(*signals_to_mute):
    """Context manager ปิด signals ชั่วคราว"""
    handlers = {}
    
    for signal in signals_to_mute:
        handlers[signal] = signal.receivers
        signal.receivers = []
    
    try:
        yield
    finally:
        for signal, signal_handlers in handlers.items():
            signal.receivers = signal_handlers

# ใช้งาน
def bulk_import_articles(articles_data):
    """Import หลาย articles โดยไม่ trigger signals"""
    with mute_signals(post_save):
        for data in articles_data:
            Article.objects.create(**data)
```

---

## 11. Signal Testing

```python
# tests/test_signals.py
from django.test import TestCase
from unittest.mock import patch, MagicMock
from django.contrib.auth import get_user_model
from .models import Article
from .signals import article_published

User = get_user_model()

class SignalTest(TestCase):
    
    def setUp(self):
        self.user = User.objects.create_user(
            username='testuser', password='pass123'
        )
    
    def test_profile_created_on_user_create(self):
        """ทดสอบ signal สร้าง profile อัตโนมัติ"""
        from .models import UserProfile
        
        new_user = User.objects.create_user(
            username='newuser', password='pass123'
        )
        
        # ตรวจสอบว่า profile ถูกสร้าง
        self.assertTrue(UserProfile.objects.filter(user=new_user).exists())
    
    @patch('myapp.signals.send_article_notification')
    def test_notification_sent_on_publish(self, mock_notify):
        """ทดสอบว่า notification ถูกส่งเมื่อเผยแพร่"""
        article = Article.objects.create(
            title='Test', slug='test',
            author=self.user, content='Content',
            status='draft'
        )
        
        # เผยแพร่
        article.status = 'published'
        article.save()
        
        # ตรวจสอบว่า notification ถูกเรียก
        mock_notify.assert_called_once_with(article)
    
    def test_custom_signal_sent(self):
        """ทดสอบ custom signal"""
        received_signals = []
        
        def handler(sender, article, publisher, **kwargs):
            received_signals.append({'article': article, 'publisher': publisher})
        
        # connect handler
        article_published.connect(handler)
        
        try:
            article = Article.objects.create(
                title='Test', slug='test-sig',
                author=self.user, content='Content'
            )
            # ส่ง signal
            article_published.send(
                sender=Article,
                article=article,
                publisher=self.user
            )
            
            self.assertEqual(len(received_signals), 1)
            self.assertEqual(received_signals[0]['article'], article)
        finally:
            # disconnect handler
            article_published.disconnect(handler)
```

---

## 12. สรุป Part 070

✅ **Signals** เป็น observer pattern สำหรับ loose coupling ระหว่าง components
✅ **pre_save/post_save** trigger ก่อน/หลัง model.save()
✅ **pre_delete/post_delete** trigger ก่อน/หลัง model.delete()
✅ **m2m_changed** trigger เมื่อ ManyToManyField เปลี่ยนแปลง
✅ **@receiver decorator** ลงทะเบียน signal receiver สะดวกกว่า connect()
✅ **Custom signals** สร้าง Signal() instance และ send() เมื่อ event เกิดขึ้น
✅ ลงทะเบียน signals ใน `AppConfig.ready()` เพื่อให้โหลดถูกเวลา
✅ ระวัง **infinite loop** - ใช้ `update()` แทน `save()` ใน signal receivers
✅ ทดสอบ signals ด้วย `unittest.mock.patch`

## ➡️ ถัดไป: Part 071 - Django Middleware

*Part 070/100+ | Python Course - Beginner to World-Class*
