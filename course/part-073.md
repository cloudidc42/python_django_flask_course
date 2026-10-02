# Part 073: Django Caching

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ cache framework ของ Django
- ตั้งค่า cache backends (Memcached, Redis)
- ใช้ per-view caching
- ใช้ template fragment caching
- ใช้ low-level cache API
- Cache invalidation strategies

---

## 1. Cache คืออะไร?

Caching เก็บผลลัพธ์ที่ใช้เวลาคำนวณนาน ไว้ใน memory เพื่อตอบกลับ request ที่เหมือนกันได้เร็วขึ้น

```
ไม่มี cache:
Request → Query DB (50ms) → Process (10ms) → Response (60ms)
Request → Query DB (50ms) → Process (10ms) → Response (60ms)

มี cache:
Request → Cache miss → Query DB → Cache → Response (60ms)  # ครั้งแรก
Request → Cache hit → Response (1ms)  # ครั้งต่อไป (60x เร็วกว่า!)
```

---

## 2. Cache Backends

```python
# settings.py

# 1. In-memory cache (default - เร็วแต่ข้ามกระบวนการไม่ได้)
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
        'LOCATION': 'unique-snowflake',
    }
}

# 2. Dummy cache (development - ไม่ cache จริง)
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.dummy.DummyCache',
    }
}

# 3. File-based cache
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.filebased.FileBasedCache',
        'LOCATION': '/var/tmp/django_cache',
    }
}

# 4. Memcached
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.PyMemcacheCache',
        'LOCATION': '127.0.0.1:11211',
    }
}

# 5. Redis (แนะนำ - เร็ว, รองรับ data types หลายอย่าง)
# pip install django-redis
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'PASSWORD': '',  # Redis password (ถ้ามี)
            'SOCKET_CONNECT_TIMEOUT': 5,
            'SOCKET_TIMEOUT': 5,
            'CONNECTION_POOL_KWARGS': {
                'max_connections': 50,
            },
            'COMPRESSOR': 'django_redis.compressors.zlib.ZlibCompressor',  # compress data
        },
        'KEY_PREFIX': 'myapp',
        'TIMEOUT': 300,  # default 5 นาที
    },
    'session': {  # cache แยกสำหรับ sessions
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/2',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        }
    }
}

# เก็บ sessions ใน Redis
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'session'
```

---

## 3. Low-level Cache API

```python
from django.core.cache import cache

# SET - เก็บข้อมูล
cache.set('my_key', 'my_value', timeout=300)  # 300 วินาที = 5 นาที
cache.set('no_expire', 'data', timeout=None)   # ไม่หมดอายุ

# GET - ดึงข้อมูล
value = cache.get('my_key')
value = cache.get('my_key', default='default_value')  # ค่า default ถ้าไม่พบ

# GET or SET - ดึงข้อมูล ถ้าไม่มีก็ set
def get_expensive_data():
    return cache.get_or_set('expensive_key', expensive_function, timeout=600)

# DELETE - ลบข้อมูล
cache.delete('my_key')

# CHECK - ตรวจสอบว่ามีหรือไม่
if cache.has_key('my_key'):
    value = cache.get('my_key')

# ตัวอย่างการใช้จริง
def get_article_stats(article_id):
    cache_key = f'article_stats_{article_id}'
    stats = cache.get(cache_key)
    
    if stats is None:
        # ไม่มีใน cache - คำนวณใหม่
        from .models import Article, Comment
        article = Article.objects.get(pk=article_id)
        stats = {
            'views': article.views_count,
            'comments': article.comments.count(),
            'likes': article.likes.count() if hasattr(article, 'likes') else 0,
        }
        # เก็บใน cache 10 นาที
        cache.set(cache_key, stats, 600)
    
    return stats
```

### Cache Operations เพิ่มเติม

```python
# SET MANY - เก็บหลายค่าพร้อมกัน
cache.set_many({
    'key1': 'value1',
    'key2': 'value2',
    'key3': 'value3',
}, timeout=300)

# GET MANY - ดึงหลายค่าพร้อมกัน
values = cache.get_many(['key1', 'key2', 'key3'])
# {'key1': 'value1', 'key2': 'value2'}  # key ที่ไม่มีจะไม่อยู่ใน result

# DELETE MANY
cache.delete_many(['key1', 'key2'])

# CLEAR - ล้าง cache ทั้งหมด (ระวัง!)
cache.clear()

# INCR/DECR - เพิ่ม/ลดตัวเลข (atomic)
cache.set('counter', 0)
cache.incr('counter')       # 1
cache.incr('counter', 5)    # 6
cache.decr('counter', 2)    # 4

# TTL (django-redis เท่านั้น)
cache.set('key', 'value', 300)
ttl = cache.ttl('key')  # วินาทีที่เหลือ

# เพิ่ม prefix
from django.core.cache import caches

# ใช้ cache อื่น
session_cache = caches['session']
session_cache.set('session_data', {...})
```

---

## 4. Per-view Caching

```python
# views.py
from django.views.decorators.cache import cache_page
from django.views.decorators.vary import vary_on_headers, vary_on_cookie

# Cache view 15 นาที
@cache_page(60 * 15)
def my_view(request):
    # ทุก request จะได้ response เดียวกันเป็นเวลา 15 นาที
    articles = Article.objects.all()
    return render(request, 'articles.html', {'articles': articles})


# Cache ต่าง language
@cache_page(60 * 15)
@vary_on_headers('Accept-Language')
def multilingual_view(request):
    return render(request, 'page.html', {})


# Cache ต่าง user (ใช้ cookies)
@cache_page(60 * 15)
@vary_on_cookie
def user_specific_view(request):
    return render(request, 'page.html', {})
```

### Cache ใน URLs

```python
# urls.py
from django.views.decorators.cache import cache_page

urlpatterns = [
    path('articles/', cache_page(60 * 15)(views.ArticleListView.as_view())),
]
```

### Cache Middleware (Site-wide caching)

```python
# settings.py
MIDDLEWARE = [
    'django.middleware.cache.UpdateCacheMiddleware',  # ต้องอยู่บนสุด
    # ...middleware อื่นๆ...
    'django.middleware.cache.FetchFromCacheMiddleware',  # ต้องอยู่ล่างสุด
]

CACHE_MIDDLEWARE_ALIAS = 'default'
CACHE_MIDDLEWARE_SECONDS = 600    # 10 นาที
CACHE_MIDDLEWARE_KEY_PREFIX = 'myapp'
```

---

## 5. Template Fragment Caching

Cache เฉพาะส่วนของ template ที่ใช้เวลา render นาน

```html
{% load cache %}

<!-- Cache template fragment 15 นาที -->
{% cache 900 article_list %}
    <!-- ส่วนนี้จะ cache 15 นาที -->
    {% for article in articles %}
    <div class="article">
        <h2>{{ article.title }}</h2>
        <p>{{ article.excerpt }}</p>
    </div>
    {% endfor %}
{% endcache %}

<!-- Cache ต่างกันตาม user -->
{% cache 900 user_profile request.user.id %}
    <div class="profile">
        <h3>{{ request.user.get_full_name }}</h3>
        <!-- ข้อมูล profile ที่ต่างกันต่อ user -->
    </div>
{% endcache %}

<!-- Cache ต่างกันตาม category -->
{% cache 600 article_list request.GET.category %}
    {% for article in articles %}
    ...
    {% endfor %}
{% endcache %}

<!-- ใช้ cache อื่น -->
{% cache 900 footer_content using="secondary" %}
    <!-- footer content -->
{% endcache %}
```

---

## 6. Cache Patterns

### Cache-Aside Pattern

```python
def get_article(article_id):
    """Cache-aside: ดึงจาก cache ก่อน ถ้าไม่มีค่อย query DB"""
    cache_key = f'article:{article_id}'
    article = cache.get(cache_key)
    
    if article is None:
        try:
            article = Article.objects.select_related('author', 'category').get(pk=article_id)
            cache.set(cache_key, article, timeout=300)
        except Article.DoesNotExist:
            return None
    
    return article
```

### Cache Invalidation

```python
# signals.py - ล้าง cache เมื่อข้อมูลเปลี่ยน
from django.db.models.signals import post_save, post_delete
from django.dispatch import receiver
from django.core.cache import cache

@receiver(post_save, sender=Article)
@receiver(post_delete, sender=Article)
def invalidate_article_cache(sender, instance, **kwargs):
    """ล้าง cache เมื่อบทความเปลี่ยน"""
    keys_to_delete = [
        f'article:{instance.pk}',
        f'article_slug:{instance.slug}',
        f'article_list',
        f'article_list_page_*',  # pattern (redis only)
    ]
    cache.delete_many(keys_to_delete)
    
    # ล้าง category cache ด้วย
    if instance.category_id:
        cache.delete(f'category:{instance.category_id}_articles')


# Django-redis: ลบ keys ด้วย pattern
from django_redis import get_redis_connection

def clear_pattern(pattern):
    """ลบ cache keys ที่ match pattern (redis only)"""
    conn = get_redis_connection('default')
    keys = conn.keys(f'*{pattern}*')
    if keys:
        conn.delete(*keys)
```

### Memoization Decorator

```python
import functools
from django.core.cache import cache

def memoize(timeout=300, key_prefix='memo'):
    """Decorator สำหรับ memoize function results"""
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            # สร้าง cache key จาก function name และ args
            cache_key = f'{key_prefix}:{func.__name__}:{hash((args, tuple(sorted(kwargs.items()))))}'
            
            result = cache.get(cache_key)
            if result is None:
                result = func(*args, **kwargs)
                cache.set(cache_key, result, timeout)
            
            return result
        
        # เพิ่ม method สำหรับล้าง cache
        def invalidate(*args, **kwargs):
            cache_key = f'{key_prefix}:{func.__name__}:{hash((args, tuple(sorted(kwargs.items()))))}'
            cache.delete(cache_key)
        
        wrapper.invalidate = invalidate
        return wrapper
    return decorator


# ใช้งาน
@memoize(timeout=600, key_prefix='stats')
def get_user_statistics(user_id):
    """คำนวณสถิติ user - cache 10 นาที"""
    from django.contrib.auth import get_user_model
    User = get_user_model()
    
    user = User.objects.get(pk=user_id)
    return {
        'articles': user.articles.count(),
        'comments': user.comments.count(),
        'total_views': user.articles.aggregate(
            total=Sum('views_count')
        )['total'] or 0,
    }

# เรียกใช้
stats = get_user_statistics(1)  # ครั้งแรก - คำนวณจริง
stats = get_user_statistics(1)  # ครั้งต่อไป - จาก cache

# ล้าง cache สำหรับ user_id=1
get_user_statistics.invalidate(1)
```

---

## 7. Cache Headers สำหรับ Browser

```python
from django.views.decorators.cache import cache_control, never_cache

# Browser cache 1 ชั่วโมง
@cache_control(max_age=3600, public=True)
def static_page(request):
    return render(request, 'static_page.html')

# ไม่ให้ browser cache
@never_cache
def sensitive_view(request):
    return render(request, 'sensitive.html')

# Private cache (เฉพาะ browser ไม่ cache ใน proxy)
@cache_control(private=True, max_age=300)
def user_dashboard(request):
    return render(request, 'dashboard.html')
```

---

## 8. สรุป Part 073

✅ **Cache Backends**: LocMem, File, Memcached, Redis - แต่ละอันเหมาะกับ use case ต่างกัน
✅ **Redis** แนะนำสำหรับ production - เร็ว, รองรับ data structures
✅ **Low-level API**: `cache.get/set/delete/get_many/set_many`
✅ **@cache_page** cache ทั้ง view response
✅ **Template fragment caching** cache เฉพาะส่วนของ template
✅ **Cache invalidation** ต้องล้าง cache เมื่อข้อมูลเปลี่ยน (signals)
✅ **Cache-aside pattern** ดึง cache ก่อน miss แล้วค่อย query DB
✅ **Cache headers** ควบคุม browser/proxy caching

## ➡️ ถัดไป: Part 074 - Django Testing

*Part 073/100+ | Python Course - Beginner to World-Class*
