# Part 56: Django URLs

## เป้าหมายของบทเรียน

- เข้าใจ URL patterns และ urlpatterns
- ใช้ `path()` และ `re_path()`
- กำหนด URL parameters ประเภทต่างๆ
- ใช้ URL namespaces
- ใช้ `reverse()` และ `get_absolute_url()`
- Include URLs จาก multiple apps
- URL converters แบบ custom

---

## 1. URL Configuration พื้นฐาน

```python
# mysite/urls.py (root URL configuration)
from django.contrib import admin
from django.urls import path, include, re_path
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    # Admin
    path('admin/', admin.site.urls),
    
    # Include URLs จาก apps
    path('blog/', include('blog.urls')),
    path('shop/', include('shop.urls')),
    path('accounts/', include('django.contrib.auth.urls')),
    path('', include('pages.urls')),
    
    # หรือ include พร้อม namespace
    path('blog/', include(('blog.urls', 'blog'))),
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

---

## 2. path() Function

```python
# blog/urls.py
from django.urls import path
from . import views

app_name = 'blog'  # namespace

urlpatterns = [
    # path(route, view, kwargs=None, name=None)
    
    # URL ไม่มี parameter
    path('', views.PostListView.as_view(), name='post_list'),
    path('about/', views.about, name='about'),
    
    # URL กับ int parameter
    path('post/<int:pk>/', views.post_detail, name='post_detail_by_id'),
    
    # URL กับ slug parameter
    path('post/<slug:slug>/', views.post_detail, name='post_detail'),
    
    # URL กับ string parameter
    path('category/<str:category_name>/', views.category_posts, name='category_posts'),
    
    # URL กับ uuid parameter
    import uuid
    path('order/<uuid:order_id>/', views.order_detail, name='order_detail'),
    
    # URL กับ path parameter (รวมถึง /)
    path('media/<path:file_path>', views.serve_file, name='serve_file'),
    
    # URL กับหลาย parameters
    path('archive/<int:year>/<int:month>/', views.archive, name='archive'),
    
    # URL กับ optional parameter ใช้ 2 paths
    path('posts/', views.post_list, name='post_list'),
    path('posts/page/<int:page>/', views.post_list, name='post_list_page'),
]
```

### Path Converters ที่มีให้ใช้

| Converter | ตัวอย่าง | ตรงกับ |
|-----------|---------|--------|
| `str` | `<str:name>` | string ที่ไม่มี `/` (default) |
| `int` | `<int:pk>` | จำนวนเต็มบวก |
| `slug` | `<slug:slug>` | ASCII letters, numbers, hyphens, underscores |
| `uuid` | `<uuid:id>` | UUID format |
| `path` | `<path:filepath>` | string รวมถึง `/` |

---

## 3. re_path() Function (Regex URLs)

```python
from django.urls import re_path
from . import views

urlpatterns = [
    # re_path ใช้ regex pattern
    
    # Match: /post/2026/10/hello-world/
    re_path(
        r'^post/(?P<year>[0-9]{4})/(?P<month>[0-9]{2})/(?P<slug>[-\w]+)/$',
        views.post_detail,
        name='post_detail_dated'
    ),
    
    # Match phone numbers: /contact/0812345678/ หรือ /contact/02-123-4567/
    re_path(
        r'^contact/(?P<phone>[\d\-]+)/$',
        views.contact_by_phone,
        name='contact_by_phone'
    ),
    
    # Match: /tag/python/ หรือ /tag/django-rest/
    re_path(
        r'^tag/(?P<tag_slug>[\w-]+)/$',
        views.tag_posts,
        name='tag_posts'
    ),
]
```

---

## 4. URL Namespaces

Namespaces ป้องกันการชนกันของ URL names เมื่อมีหลาย apps

### Application Namespace

```python
# blog/urls.py
from django.urls import path
from . import views

# กำหนด app_name สำหรับ application namespace
app_name = 'blog'

urlpatterns = [
    path('', views.post_list, name='post_list'),
    path('<slug:slug>/', views.post_detail, name='post_detail'),
    path('create/', views.post_create, name='post_create'),
    path('<slug:slug>/edit/', views.post_update, name='post_update'),
    path('<slug:slug>/delete/', views.post_delete, name='post_delete'),
]
```

```python
# shop/urls.py
app_name = 'shop'

urlpatterns = [
    path('', views.product_list, name='product_list'),
    path('<slug:slug>/', views.product_detail, name='product_detail'),
]
```

```python
# mysite/urls.py
urlpatterns = [
    path('blog/', include('blog.urls')),   # blog namespace
    path('shop/', include('shop.urls')),   # shop namespace
]
```

### Instance Namespace

```python
# mysite/urls.py
urlpatterns = [
    # กำหนด instance namespace แยกจาก application namespace
    path('blog/', include(('blog.urls', 'blog'), namespace='main-blog')),
    path('admin-blog/', include(('blog.urls', 'blog'), namespace='admin-blog')),
]
```

### ใช้ Namespace ใน Templates

```html
<!-- template -->

<!-- blog:post_list = app_name:url_name -->
<a href="{% url 'blog:post_list' %}">บทความทั้งหมด</a>
<a href="{% url 'blog:post_detail' slug=post.slug %}">{{ post.title }}</a>
<a href="{% url 'blog:post_create' %}">เขียนบทความ</a>

<!-- shop namespace -->
<a href="{% url 'shop:product_list' %}">สินค้าทั้งหมด</a>
<a href="{% url 'shop:product_detail' slug=product.slug %}">{{ product.name }}</a>

<!-- built-in auth URLs -->
<a href="{% url 'login' %}">เข้าสู่ระบบ</a>
<a href="{% url 'logout' %}">ออกจากระบบ</a>
<a href="{% url 'password_change' %}">เปลี่ยนรหัสผ่าน</a>
```

---

## 5. reverse() Function

`reverse()` แปลง URL name กลับเป็น URL string

```python
# views.py
from django.urls import reverse
from django.shortcuts import redirect


def some_view(request):
    # reverse คืน URL string
    url = reverse('blog:post_list')
    # ผลลัพธ์: '/blog/'
    
    # reverse กับ args
    url = reverse('blog:post_detail', args=['my-post-slug'])
    # ผลลัพธ์: '/blog/my-post-slug/'
    
    # reverse กับ kwargs
    url = reverse('blog:post_detail', kwargs={'slug': 'my-post-slug'})
    # ผลลัพธ์: '/blog/my-post-slug/'
    
    # reverse กับ current_app (instance namespace)
    url = reverse('blog:post_list', current_app='admin-blog')
    
    # ใช้ใน redirect
    return redirect(reverse('blog:post_list'))
    # หรือสั้นกว่า:
    return redirect('blog:post_list')
    return redirect('blog:post_detail', slug='my-post')


# reverse_lazy - สำหรับ class-level attribute
from django.urls import reverse_lazy

class PostDeleteView(DeleteView):
    # ใช้ reverse_lazy แทน reverse เพราะ evaluate ตอน class load
    success_url = reverse_lazy('blog:post_list')
```

---

## 6. get_absolute_url()

```python
# models.py
from django.db import models
from django.urls import reverse


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    
    def get_absolute_url(self):
        """คืน URL สำหรับเข้าถึง Post นี้"""
        return reverse('blog:post_detail', kwargs={'slug': self.slug})
    
    # ถ้ามีหลาย parameters
    # def get_absolute_url(self):
    #     return reverse('blog:post_dated', kwargs={
    #         'year': self.created_at.year,
    #         'month': self.created_at.month,
    #         'slug': self.slug,
    #     })


class Product(models.Model):
    name = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    category = models.ForeignKey('Category', on_delete=models.CASCADE)
    
    def get_absolute_url(self):
        return reverse('shop:product_detail', kwargs={
            'category_slug': self.category.slug,
            'product_slug': self.slug,
        })
```

```html
<!-- template -->
<!-- ใช้ get_absolute_url() ใน template -->
<a href="{{ post.get_absolute_url }}">{{ post.title }}</a>
<a href="{{ product.get_absolute_url }}">{{ product.name }}</a>
```

---

## 7. URL Patterns แบบต่างๆ

### URL สำหรับ Blog App

```python
# blog/urls.py
from django.urls import path, re_path
from . import views

app_name = 'blog'

urlpatterns = [
    # รายการบทความ
    path('', views.PostListView.as_view(), name='post_list'),
    
    # รายละเอียดบทความ
    path('<slug:slug>/', views.PostDetailView.as_view(), name='post_detail'),
    
    # สร้าง แก้ไข ลบ
    path('create/', views.PostCreateView.as_view(), name='post_create'),
    path('<slug:slug>/edit/', views.PostUpdateView.as_view(), name='post_update'),
    path('<slug:slug>/delete/', views.PostDeleteView.as_view(), name='post_delete'),
    
    # Comments
    path('<slug:post_slug>/comment/', views.add_comment, name='add_comment'),
    path('comment/<int:pk>/delete/', views.delete_comment, name='delete_comment'),
    
    # Archive
    path('archive/<int:year>/', views.YearArchive.as_view(), name='year_archive'),
    path('archive/<int:year>/<int:month>/', views.MonthArchive.as_view(), name='month_archive'),
    
    # Category และ Tag
    path('category/<slug:slug>/', views.CategoryDetail.as_view(), name='category_detail'),
    path('tag/<slug:slug>/', views.TagDetail.as_view(), name='tag_detail'),
    
    # Author
    path('author/<str:username>/', views.AuthorPosts.as_view(), name='author_posts'),
    
    # Feed
    path('feed/', views.LatestPostsFeed(), name='post_feed'),
    
    # API endpoints
    path('api/posts/', views.PostListAPI.as_view(), name='api_post_list'),
    path('api/posts/<int:pk>/', views.PostDetailAPI.as_view(), name='api_post_detail'),
    
    # Search
    path('search/', views.search, name='search'),
]
```

### URL สำหรับ Shop App

```python
# shop/urls.py
from django.urls import path
from . import views

app_name = 'shop'

urlpatterns = [
    # หน้าแรก shop
    path('', views.ProductListView.as_view(), name='product_list'),
    
    # หมวดหมู่
    path('category/<slug:slug>/', views.CategoryView.as_view(), name='category'),
    
    # สินค้า - path หลาย levels
    path('<slug:category_slug>/<slug:product_slug>/', 
         views.ProductDetailView.as_view(), 
         name='product_detail'),
    
    # ตะกร้า
    path('cart/', views.CartView.as_view(), name='cart'),
    path('cart/add/<int:product_id>/', views.cart_add, name='cart_add'),
    path('cart/remove/<int:item_id>/', views.cart_remove, name='cart_remove'),
    path('cart/update/', views.cart_update, name='cart_update'),
    
    # Checkout
    path('checkout/', views.CheckoutView.as_view(), name='checkout'),
    path('checkout/payment/', views.PaymentView.as_view(), name='payment'),
    
    # Orders
    path('orders/', views.OrderListView.as_view(), name='order_list'),
    path('orders/<uuid:order_id>/', views.OrderDetailView.as_view(), name='order_detail'),
]
```

---

## 8. Custom Path Converters

```python
# blog/converters.py
class FourDigitYearConverter:
    """
    URL converter สำหรับปี 4 หลัก
    ตรงกับ /archive/2026/
    """
    # regex pattern
    regex = '[0-9]{4}'
    
    def to_python(self, value):
        """แปลง string จาก URL เป็น Python object"""
        return int(value)
    
    def to_url(self, value):
        """แปลง Python object กลับเป็น string ใน URL"""
        return '%04d' % value


class NegativeIntConverter:
    """
    URL converter รองรับจำนวนเต็มลบ
    """
    regex = '-?[0-9]+'
    
    def to_python(self, value):
        return int(value)
    
    def to_url(self, value):
        return str(value)


# ลงทะเบียน converters
# urls.py
from django.urls import path, register_converter
from . import converters, views

register_converter(converters.FourDigitYearConverter, 'yyyy')
register_converter(converters.NegativeIntConverter, 'negint')

urlpatterns = [
    # ใช้ custom converter
    path('archive/<yyyy:year>/', views.year_archive, name='year_archive'),
    path('offset/<negint:offset>/', views.offset_view, name='offset'),
]
```

---

## 9. Error Handling URLs

```python
# mysite/urls.py
from django.conf.urls import handler400, handler403, handler404, handler500

handler400 = 'mysite.views.bad_request'
handler403 = 'mysite.views.permission_denied'
handler404 = 'mysite.views.page_not_found'
handler500 = 'mysite.views.server_error'
```

```python
# mysite/views.py
from django.shortcuts import render


def page_not_found(request, exception):
    """404 Error Handler"""
    return render(request, 'errors/404.html', status=404)


def server_error(request):
    """500 Error Handler"""
    return render(request, 'errors/500.html', status=500)


def permission_denied(request, exception):
    """403 Error Handler"""
    return render(request, 'errors/403.html', status=403)


def bad_request(request, exception):
    """400 Error Handler"""
    return render(request, 'errors/400.html', status=400)
```

```html
<!-- templates/errors/404.html -->
{% extends 'base.html' %}

{% block title %}404 - หน้านี้ไม่มี{% endblock %}

{% block content %}
<div class="text-center py-5">
    <h1 class="display-1 text-muted">404</h1>
    <h2>ไม่พบหน้าที่คุณต้องการ</h2>
    <p class="text-muted">URL ที่คุณพิมพ์อาจไม่ถูกต้อง หรือหน้านี้ถูกลบออกไปแล้ว</p>
    <a href="{% url 'blog:post_list' %}" class="btn btn-primary">กลับหน้าแรก</a>
</div>
{% endblock %}
```

---

## 10. URL ขั้นสูง

### Django REST Framework Router

```python
# หากใช้ DRF
from rest_framework.routers import DefaultRouter
from . import views

router = DefaultRouter()
router.register(r'posts', views.PostViewSet, basename='post')
router.register(r'categories', views.CategoryViewSet, basename='category')

urlpatterns = [
    path('api/', include(router.urls)),
]
# สร้าง URLs อัตโนมัติ:
# /api/posts/ -> list, create
# /api/posts/{pk}/ -> retrieve, update, delete
# /api/categories/ -> list, create
# /api/categories/{pk}/ -> retrieve, update, delete
```

### URL กับ Extra kwargs

```python
# ส่ง extra data ไปยัง view
urlpatterns = [
    path('archive/', views.archive, {'paginate_by': 20}, name='archive'),
    # views.archive จะได้ request, paginate_by=20
]
```

---

## 11. ตัวอย่างเต็ม: Blog URL Structure

```
/                           -> หน้าแรก (pages:home)
/blog/                      -> รายการบทความ (blog:post_list)
/blog/create/               -> สร้างบทความ (blog:post_create)
/blog/search/               -> ค้นหา (blog:search)
/blog/my-first-post/        -> รายละเอียดบทความ (blog:post_detail)
/blog/my-first-post/edit/   -> แก้ไขบทความ (blog:post_update)
/blog/my-first-post/delete/ -> ลบบทความ (blog:post_delete)
/blog/my-first-post/comment/ -> เพิ่มคอมเมนต์ (blog:add_comment)
/blog/category/python/      -> บทความในหมวด (blog:category_detail)
/blog/tag/django/           -> บทความในแท็ก (blog:tag_detail)
/blog/archive/2026/         -> บทความปี 2026 (blog:year_archive)
/blog/archive/2026/10/      -> บทความ ต.ค. 2026 (blog:month_archive)
/blog/author/somchai/       -> บทความของ somchai (blog:author_posts)

/shop/                      -> รายการสินค้า (shop:product_list)
/shop/electronics/          -> หมวดหมู่ (shop:category)
/shop/electronics/laptop/   -> รายละเอียดสินค้า (shop:product_detail)
/shop/cart/                 -> ตะกร้า (shop:cart)
/shop/checkout/             -> checkout (shop:checkout)
/shop/orders/               -> รายการคำสั่งซื้อ (shop:order_list)

/accounts/login/            -> เข้าสู่ระบบ
/accounts/logout/           -> ออกจากระบบ
/accounts/register/         -> สมัครสมาชิก
/accounts/password/change/  -> เปลี่ยนรหัสผ่าน

/admin/                     -> Django Admin
```

```python
# mysite/urls.py
from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static


urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('pages.urls')),
    path('blog/', include('blog.urls')),
    path('shop/', include('shop.urls')),
    path('accounts/', include('accounts.urls')),
    path('accounts/', include('django.contrib.auth.urls')),
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)

# Error handlers (ใช้งานเมื่อ DEBUG=False)
handler404 = 'mysite.views.page_not_found'
handler500 = 'mysite.views.server_error'
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: URL Design
ออกแบบ URL structure สำหรับ Forum website:
- Forum categories
- Threads ในแต่ละ category
- Posts ในแต่ละ thread
- User profiles
- Search

### แบบฝึกหัดที่ 2: Custom Converter
สร้าง custom path converter:
- `ThaiDateConverter` - รับวันที่รูปแบบ `dd-mm-yyyy` เช่น `/archive/01-10-2026/`
- แปลงเป็น Python `datetime.date` object
- ใช้ใน archive view

### แบบฝึกหัดที่ 3: URL Testing
เขียน unit tests สำหรับ URLs:
```python
from django.test import TestCase
from django.urls import reverse, resolve

class BlogURLTests(TestCase):
    def test_post_list_url(self):
        url = reverse('blog:post_list')
        self.assertEqual(url, '/blog/')
        
    def test_post_list_resolves(self):
        resolver = resolve('/blog/')
        self.assertEqual(resolver.func.view_class, PostListView)
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- urlpatterns และ path() function
- Path converters (str, int, slug, uuid, path)
- re_path() สำหรับ regex patterns
- URL namespaces (application และ instance)
- reverse() และ reverse_lazy()
- get_absolute_url() ใน models
- Custom path converters
- Error handling URLs

---

## บทถัดไป

➡️ **[Part 57: Django Admin](part-057.md)** - เรียนรู้การ customize Django Admin interface
