# Part 52: Django Models

## เป้าหมายของบทเรียน

- สร้าง Model และกำหนด Field types ต่างๆ
- เข้าใจ Field options ทุกประเภท
- ใช้ `__str__` method และ Meta class
- ใช้ Django ORM สำหรับ CRUD operations
- ใช้ QuerySet API อย่างมีประสิทธิภาพ
- จัดการ Related objects (ForeignKey, ManyToMany)
- ตัวอย่าง Blog application models

---

## 1. Django Model คืออะไร?

Model เป็น Python class ที่สืบทอดจาก `django.db.models.Model` ซึ่งแต่ละ attribute ของ class จะแมปกับ column ในฐานข้อมูล

```python
# blog/models.py
from django.db import models

# ตัวอย่าง Model อย่างง่าย
class Article(models.Model):
    # แต่ละ attribute = column ในฐานข้อมูล
    title = models.CharField(max_length=200)   # VARCHAR(200)
    content = models.TextField()               # TEXT
    created_at = models.DateTimeField(auto_now_add=True)  # DATETIME
    is_published = models.BooleanField(default=False)     # BOOLEAN
```

Django จะสร้าง SQL ให้อัตโนมัติ:
```sql
CREATE TABLE "blog_article" (
    "id" integer NOT NULL PRIMARY KEY AUTOINCREMENT,
    "title" varchar(200) NOT NULL,
    "content" text NOT NULL,
    "created_at" datetime NOT NULL,
    "is_published" bool NOT NULL
);
```

---

## 2. Field Types ทุกประเภท

### Text Fields

```python
# blog/models.py
from django.db import models

class TextFieldExamples(models.Model):
    # CharField - string สั้น ต้องระบุ max_length
    name = models.CharField(max_length=100)
    
    # TextField - text ยาว ไม่จำกัด
    description = models.TextField()
    
    # EmailField - validate email format
    email = models.EmailField(max_length=254)
    
    # URLField - validate URL format
    website = models.URLField(max_length=200)
    
    # SlugField - สำหรับ URL-friendly strings
    slug = models.SlugField(max_length=100, unique=True)
    
    # UUIDField - สำหรับ UUID
    import uuid
    uuid = models.UUIDField(default=uuid.uuid4, editable=False)
    
    # IPAddressField - IP address
    ip_address = models.GenericIPAddressField(null=True, blank=True)
```

### Numeric Fields

```python
class NumericFieldExamples(models.Model):
    # IntegerField - จำนวนเต็ม (-2147483648 ถึง 2147483647)
    age = models.IntegerField()
    
    # PositiveIntegerField - จำนวนเต็มบวก
    quantity = models.PositiveIntegerField()
    
    # BigIntegerField - จำนวนเต็มขนาดใหญ่
    big_number = models.BigIntegerField()
    
    # SmallIntegerField - จำนวนเต็มขนาดเล็ก (-32768 ถึง 32767)
    small_number = models.SmallIntegerField()
    
    # FloatField - floating point
    temperature = models.FloatField()
    
    # DecimalField - ทศนิยมแม่นยำสูง (สำหรับเงิน)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    
    # PositiveSmallIntegerField
    rating = models.PositiveSmallIntegerField()
```

### Date/Time Fields

```python
class DateTimeFieldExamples(models.Model):
    # DateField - วันที่เท่านั้น
    birth_date = models.DateField()
    
    # TimeField - เวลาเท่านั้น
    opening_time = models.TimeField()
    
    # DateTimeField - วันที่และเวลา
    created_at = models.DateTimeField()
    
    # auto_now_add - กำหนดค่าเมื่อสร้างครั้งแรก
    created = models.DateTimeField(auto_now_add=True)
    
    # auto_now - อัปเดตทุกครั้งที่ save
    updated = models.DateTimeField(auto_now=True)
    
    # DurationField - ระยะเวลา
    duration = models.DurationField()
```

### Boolean Fields

```python
class BooleanFieldExamples(models.Model):
    # BooleanField - True/False
    is_active = models.BooleanField(default=True)
    
    # NullBooleanField (deprecated ใน Django 4.0)
    # ใช้ BooleanField(null=True) แทน
    has_premium = models.BooleanField(null=True, blank=True)
```

### File Fields

```python
class FileFieldExamples(models.Model):
    # FileField - upload ไฟล์
    document = models.FileField(upload_to='documents/')
    
    # ImageField - upload รูปภาพ (ต้องติดตั้ง Pillow)
    avatar = models.ImageField(
        upload_to='avatars/',
        null=True,
        blank=True
    )
    
    # upload_to รองรับ callable
    def user_directory_path(instance, filename):
        """สร้าง path สำหรับ upload ตาม user id"""
        return f'user_{instance.user.id}/avatars/{filename}'
    
    profile_picture = models.ImageField(
        upload_to=user_directory_path,
        null=True,
        blank=True
    )
```

### Relationship Fields

```python
class Category(models.Model):
    name = models.CharField(max_length=100)

class Tag(models.Model):
    name = models.CharField(max_length=50)

class Post(models.Model):
    title = models.CharField(max_length=200)
    
    # ForeignKey - many-to-one (หลาย Post อยู่ใน 1 Category)
    category = models.ForeignKey(
        Category,
        on_delete=models.CASCADE,     # ลบ Category -> ลบ Posts ด้วย
        related_name='posts'           # ชื่อสำหรับ reverse relation
    )
    
    # ManyToManyField - หลาย Post มี หลาย Tags
    tags = models.ManyToManyField(
        Tag,
        blank=True,                    # optional
        related_name='posts'
    )
    
    # OneToOneField - one-to-one
    # (ใช้สำหรับ extend User model เป็นต้น)

class UserProfile(models.Model):
    from django.contrib.auth.models import User
    user = models.OneToOneField(
        User,
        on_delete=models.CASCADE,
        related_name='profile'
    )
    bio = models.TextField(blank=True)
```

**on_delete options:**
- `CASCADE` - ลบ related objects ด้วย
- `PROTECT` - ป้องกันการลบถ้ายังมี related objects
- `SET_NULL` - กำหนดเป็น NULL (ต้องมี null=True)
- `SET_DEFAULT` - กำหนดเป็น default value
- `DO_NOTHING` - ไม่ทำอะไร (อาจเกิด IntegrityError)

### JSON Field (Django 3.1+)

```python
class JSONFieldExamples(models.Model):
    # JSONField - เก็บข้อมูล JSON
    metadata = models.JSONField(default=dict)
    settings = models.JSONField(null=True, blank=True)
```

---

## 3. Field Options

```python
class FieldOptionsExample(models.Model):
    # null=True - อนุญาตค่า NULL ในฐานข้อมูล
    middle_name = models.CharField(max_length=50, null=True)
    
    # blank=True - อนุญาตค่าว่างในการ validate (forms)
    bio = models.TextField(blank=True)
    
    # null=True + blank=True - optional field
    phone = models.CharField(max_length=20, null=True, blank=True)
    
    # default - ค่าเริ่มต้น
    status = models.CharField(max_length=20, default='draft')
    created_count = models.IntegerField(default=0)
    
    # unique - ต้องไม่ซ้ำในตาราง
    username = models.CharField(max_length=50, unique=True)
    
    # unique_together (ใน Meta class)
    # db_index=True - สร้าง database index เพื่อเร่งความเร็ว query
    email = models.EmailField(db_index=True)
    
    # choices - กำหนด choices สำหรับ dropdown
    STATUS_CHOICES = [
        ('draft', 'แบบร่าง'),
        ('published', 'เผยแพร่แล้ว'),
        ('archived', 'เก็บเข้าคลัง'),
    ]
    status = models.CharField(
        max_length=20,
        choices=STATUS_CHOICES,
        default='draft'
    )
    
    # verbose_name - ชื่อที่แสดงใน admin
    created_at = models.DateTimeField(
        auto_now_add=True,
        verbose_name='วันที่สร้าง'
    )
    
    # help_text - คำแนะนำในฟอร์ม
    password = models.CharField(
        max_length=128,
        help_text='รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร'
    )
    
    # editable=False - ไม่แสดงในฟอร์ม
    auto_field = models.IntegerField(editable=False, default=0)
    
    # primary_key=True - กำหนดเป็น primary key
    # (ถ้าไม่กำหนด Django จะสร้าง id autoincrement ให้)
    # custom_id = models.CharField(max_length=10, primary_key=True)
```

### ใช้ Choices แบบ Enum (แนะนำ)

```python
from django.db import models


class Article(models.Model):
    # ใช้ TextChoices หรือ IntegerChoices แทน list of tuples
    class Status(models.TextChoices):
        DRAFT = 'draft', 'แบบร่าง'
        PUBLISHED = 'published', 'เผยแพร่'
        ARCHIVED = 'archived', 'เก็บเข้าคลัง'
    
    class Priority(models.IntegerChoices):
        LOW = 1, 'ต่ำ'
        MEDIUM = 2, 'ปานกลาง'
        HIGH = 3, 'สูง'
    
    title = models.CharField(max_length=200)
    status = models.CharField(
        max_length=20,
        choices=Status.choices,
        default=Status.DRAFT
    )
    priority = models.IntegerField(
        choices=Priority.choices,
        default=Priority.MEDIUM
    )
    
    def is_published(self):
        """ตรวจสอบว่าบทความเผยแพร่แล้วหรือยัง"""
        return self.status == self.Status.PUBLISHED
```

---

## 4. __str__ Method และ Meta Class

### __str__ Method

```python
from django.db import models
from django.utils import timezone


class Category(models.Model):
    name = models.CharField(max_length=100, verbose_name='ชื่อหมวดหมู่')
    slug = models.SlugField(unique=True)
    
    def __str__(self):
        """แสดงชื่อ object ใน admin และ shell"""
        return self.name


class Tag(models.Model):
    name = models.CharField(max_length=50)
    
    def __str__(self):
        return f'#{self.name}'  # แสดงเป็น #python, #django เป็นต้น
```

### Meta Class

```python
class Post(models.Model):
    title = models.CharField(max_length=200, verbose_name='หัวข้อ')
    slug = models.SlugField(unique=True)
    content = models.TextField(verbose_name='เนื้อหา')
    category = models.ForeignKey(Category, on_delete=models.CASCADE)
    tags = models.ManyToManyField(Tag, blank=True)
    author = models.ForeignKey(
        'auth.User',
        on_delete=models.CASCADE,
        related_name='posts'
    )
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    published_at = models.DateTimeField(null=True, blank=True)
    is_featured = models.BooleanField(default=False)
    views_count = models.PositiveIntegerField(default=0)
    
    class Meta:
        # การเรียงลำดับ default
        ordering = ['-created_at']  # - หมายถึง descending
        
        # ชื่อที่แสดงใน admin (singular)
        verbose_name = 'บทความ'
        
        # ชื่อที่แสดงใน admin (plural)
        verbose_name_plural = 'บทความทั้งหมด'
        
        # ชื่อ table ในฐานข้อมูล (default: app_model)
        db_table = 'blog_posts'
        
        # ป้องกันการ insert/update ซ้ำ
        unique_together = [['slug', 'category']]  # deprecated, ใช้ constraints
        
        # Django 2.2+: ใช้ constraints แทน
        constraints = [
            models.UniqueConstraint(
                fields=['slug', 'author'],
                name='unique_slug_per_author'
            )
        ]
        
        # สร้าง index เพื่อเร่งความเร็ว
        indexes = [
            models.Index(fields=['created_at'], name='post_created_idx'),
            models.Index(fields=['is_featured', '-created_at']),
        ]
    
    def __str__(self):
        return self.title
    
    def get_absolute_url(self):
        """URL สำหรับเข้าถึง post นี้"""
        from django.urls import reverse
        return reverse('blog:post_detail', kwargs={'slug': self.slug})
    
    def publish(self):
        """เผยแพร่บทความ"""
        self.published_at = timezone.now()
        self.save()
    
    def increment_views(self):
        """เพิ่มจำนวนการดู"""
        self.views_count += 1
        self.save(update_fields=['views_count'])  # save เฉพาะ field ที่เปลี่ยน
    
    @property
    def reading_time(self):
        """คำนวณเวลาอ่านโดยประมาณ (นาที)"""
        words = len(self.content.split())
        return max(1, round(words / 200))  # สมมติ อ่าน 200 คำต่อนาที
```

---

## 5. Blog Models ตัวอย่างแบบสมบูรณ์

```python
# blog/models.py
from django.db import models
from django.contrib.auth.models import User
from django.urls import reverse
from django.utils import timezone
from django.utils.text import slugify


class Category(models.Model):
    """หมวดหมู่บทความ"""
    name = models.CharField(max_length=100, verbose_name='ชื่อหมวดหมู่')
    slug = models.SlugField(max_length=100, unique=True)
    description = models.TextField(blank=True, verbose_name='คำอธิบาย')
    color = models.CharField(
        max_length=7,
        default='#007bff',
        help_text='รหัสสี Hex เช่น #007bff'
    )
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        ordering = ['name']
        verbose_name = 'หมวดหมู่'
        verbose_name_plural = 'หมวดหมู่ทั้งหมด'
    
    def __str__(self):
        return self.name
    
    def save(self, *args, **kwargs):
        """สร้าง slug อัตโนมัติจาก name"""
        if not self.slug:
            self.slug = slugify(self.name)
        super().save(*args, **kwargs)
    
    def get_absolute_url(self):
        return reverse('blog:category_detail', kwargs={'slug': self.slug})


class Tag(models.Model):
    """แท็กบทความ"""
    name = models.CharField(max_length=50, unique=True, verbose_name='ชื่อแท็ก')
    slug = models.SlugField(max_length=50, unique=True)
    
    class Meta:
        ordering = ['name']
        verbose_name = 'แท็ก'
        verbose_name_plural = 'แท็กทั้งหมด'
    
    def __str__(self):
        return self.name
    
    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.name)
        super().save(*args, **kwargs)


class Post(models.Model):
    """บทความ"""
    
    class Status(models.TextChoices):
        DRAFT = 'draft', 'แบบร่าง'
        PUBLISHED = 'published', 'เผยแพร่'
        ARCHIVED = 'archived', 'เก็บเข้าคลัง'
    
    # ข้อมูลพื้นฐาน
    title = models.CharField(max_length=200, verbose_name='หัวข้อ')
    slug = models.SlugField(max_length=200, unique=True, verbose_name='Slug')
    excerpt = models.TextField(
        max_length=500,
        blank=True,
        verbose_name='บทคัดย่อ',
        help_text='สรุปสั้นๆ ของบทความ (ไม่เกิน 500 ตัวอักษร)'
    )
    content = models.TextField(verbose_name='เนื้อหา')
    
    # ความสัมพันธ์
    author = models.ForeignKey(
        User,
        on_delete=models.CASCADE,
        related_name='posts',
        verbose_name='ผู้เขียน'
    )
    category = models.ForeignKey(
        Category,
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name='posts',
        verbose_name='หมวดหมู่'
    )
    tags = models.ManyToManyField(
        Tag,
        blank=True,
        related_name='posts',
        verbose_name='แท็ก'
    )
    
    # รูปภาพ
    featured_image = models.ImageField(
        upload_to='blog/images/%Y/%m/',
        null=True,
        blank=True,
        verbose_name='รูปภาพหลัก'
    )
    
    # สถานะและวันที่
    status = models.CharField(
        max_length=20,
        choices=Status.choices,
        default=Status.DRAFT,
        verbose_name='สถานะ'
    )
    is_featured = models.BooleanField(default=False, verbose_name='บทความแนะนำ')
    created_at = models.DateTimeField(auto_now_add=True, verbose_name='วันที่สร้าง')
    updated_at = models.DateTimeField(auto_now=True, verbose_name='วันที่แก้ไข')
    published_at = models.DateTimeField(
        null=True,
        blank=True,
        verbose_name='วันที่เผยแพร่'
    )
    
    # สถิติ
    views_count = models.PositiveIntegerField(default=0, verbose_name='จำนวนครั้งที่ดู')
    
    class Meta:
        ordering = ['-published_at', '-created_at']
        verbose_name = 'บทความ'
        verbose_name_plural = 'บทความทั้งหมด'
        indexes = [
            models.Index(fields=['-created_at']),
            models.Index(fields=['status', '-published_at']),
            models.Index(fields=['author', 'status']),
        ]
    
    def __str__(self):
        return f'{self.title} ({self.get_status_display()})'
    
    def save(self, *args, **kwargs):
        """สร้าง slug อัตโนมัติ"""
        if not self.slug:
            self.slug = slugify(self.title)
        super().save(*args, **kwargs)
    
    def get_absolute_url(self):
        return reverse('blog:post_detail', kwargs={'slug': self.slug})
    
    def publish(self):
        """เผยแพร่บทความ"""
        self.status = self.Status.PUBLISHED
        self.published_at = timezone.now()
        self.save()
    
    @property
    def is_published(self):
        """ตรวจสอบว่าเผยแพร่แล้วหรือยัง"""
        return self.status == self.Status.PUBLISHED
    
    @property
    def reading_time(self):
        """เวลาอ่านโดยประมาณ (นาที)"""
        words = len(self.content.split())
        minutes = round(words / 200)
        return max(1, minutes)
    
    def increment_views(self):
        """เพิ่มจำนวน views"""
        Post.objects.filter(pk=self.pk).update(views_count=models.F('views_count') + 1)


class Comment(models.Model):
    """ความคิดเห็น"""
    post = models.ForeignKey(
        Post,
        on_delete=models.CASCADE,
        related_name='comments',
        verbose_name='บทความ'
    )
    author = models.ForeignKey(
        User,
        on_delete=models.CASCADE,
        related_name='comments',
        verbose_name='ผู้แสดงความคิดเห็น'
    )
    content = models.TextField(verbose_name='ความคิดเห็น')
    parent = models.ForeignKey(
        'self',  # self-referential FK สำหรับ nested comments
        on_delete=models.CASCADE,
        null=True,
        blank=True,
        related_name='replies',
        verbose_name='ตอบกลับ'
    )
    is_approved = models.BooleanField(default=True, verbose_name='อนุมัติแล้ว')
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        ordering = ['created_at']
        verbose_name = 'ความคิดเห็น'
        verbose_name_plural = 'ความคิดเห็นทั้งหมด'
    
    def __str__(self):
        return f'ความคิดเห็นของ {self.author} ใน {self.post}'
    
    @property
    def is_reply(self):
        """ตรวจสอบว่าเป็นการตอบกลับหรือไม่"""
        return self.parent is not None
```

---

## 6. Django ORM Queries

### การสร้างข้อมูล (Create)

```python
# ใน Django shell: python manage.py shell
from blog.models import Category, Tag, Post
from django.contrib.auth.models import User

# Method 1: สร้างและ save แยกกัน
category = Category()
category.name = 'Python'
category.slug = 'python'
category.save()

# Method 2: ส่ง kwargs ใน constructor
category = Category(name='Django', slug='django')
category.save()

# Method 3: create() - สร้างและ save ในขั้นตอนเดียว
category = Category.objects.create(
    name='Machine Learning',
    slug='machine-learning'
)

# สร้าง User ก่อน
user = User.objects.create_user(
    username='somchai',
    email='somchai@example.com',
    password='password123'
)

# สร้าง Post
post = Post.objects.create(
    title='แนะนำ Django Framework',
    slug='intro-django-framework',
    content='Django เป็น web framework ที่ยอดเยี่ยม...',
    author=user,
    category=category,
)

# เพิ่ม ManyToMany
tag1 = Tag.objects.create(name='Django', slug='django')
tag2 = Tag.objects.create(name='Python', slug='python')

post.tags.add(tag1, tag2)              # เพิ่มหลาย tags ครั้งเดียว
post.tags.add(tag1)                    # เพิ่มทีละตัว
post.tags.remove(tag2)                 # ลบออก
post.tags.set([tag1])                  # กำหนดให้มีแค่ tag1
post.tags.clear()                      # ลบทั้งหมด
```

### การอ่านข้อมูล (Read)

```python
# ดึง object เดียว
post = Post.objects.get(pk=1)          # ได้ object หรือ raise DoesNotExist
post = Post.objects.get(slug='intro-django')

# ดึง object เดียว แต่ไม่ raise exception
from django.shortcuts import get_object_or_404
post = get_object_or_404(Post, slug='intro-django')  # คืน 404 ถ้าไม่เจอ

# ดึงทั้งหมด
posts = Post.objects.all()

# กรอง
published_posts = Post.objects.filter(status='published')
featured = Post.objects.filter(is_featured=True)

# กรองหลายเงื่อนไข (AND)
posts = Post.objects.filter(
    status='published',
    is_featured=True
)

# ยกเว้น (NOT)
posts = Post.objects.exclude(status='archived')

# ตรวจสอบว่ามีอยู่หรือไม่
exists = Post.objects.filter(slug='intro-django').exists()

# นับจำนวน
count = Post.objects.filter(status='published').count()

# ดึง values บางส่วน (เร็วกว่า)
titles = Post.objects.values('title', 'slug')
# [{'title': 'post1', 'slug': 'post1'}, ...]

# ดึงแค่ค่า (list of tuples)
title_list = Post.objects.values_list('title', flat=True)
# ['post1', 'post2', ...]
```

### Field Lookups

```python
# Exact match (default)
Post.objects.filter(status='published')
Post.objects.filter(status__exact='published')  # เหมือนกัน

# Case-insensitive
Post.objects.filter(title__iexact='hello world')

# Contains (LIKE '%value%')
Post.objects.filter(title__contains='Django')
Post.objects.filter(title__icontains='django')  # case-insensitive

# Starts with / Ends with
Post.objects.filter(title__startswith='แนะนำ')
Post.objects.filter(title__endswith='Framework')

# In list
Post.objects.filter(status__in=['published', 'featured'])

# Comparison
Post.objects.filter(views_count__gt=100)   # greater than
Post.objects.filter(views_count__gte=100)  # greater than or equal
Post.objects.filter(views_count__lt=100)   # less than
Post.objects.filter(views_count__lte=100)  # less than or equal

# Range
Post.objects.filter(created_at__range=('2025-01-01', '2025-12-31'))

# Date parts
Post.objects.filter(created_at__year=2025)
Post.objects.filter(created_at__month=10)
Post.objects.filter(created_at__day=1)

# NULL check
Post.objects.filter(published_at__isnull=True)    # IS NULL
Post.objects.filter(published_at__isnull=False)   # IS NOT NULL

# Relationship traversal (double underscore)
# ดู posts ของ category ที่ชื่อ 'Python'
Post.objects.filter(category__name='Python')

# ดู posts ของ author ที่ชื่อ 'somchai'
Post.objects.filter(author__username='somchai')
```

### การอัปเดตข้อมูล (Update)

```python
# อัปเดต object เดียว
post = Post.objects.get(pk=1)
post.title = 'หัวข้อใหม่'
post.save()

# อัปเดต field เฉพาะ (ดีกว่า save() ทั้งหมด)
post.save(update_fields=['title', 'updated_at'])

# อัปเดตหลาย objects ครั้งเดียว (เร็วมาก)
Post.objects.filter(status='draft').update(status='published')

# ใช้ F() expression เพื่อ atomic update
from django.db.models import F
Post.objects.filter(pk=1).update(views_count=F('views_count') + 1)

# update_or_create
post, created = Post.objects.update_or_create(
    slug='test-post',
    defaults={
        'title': 'Test Post Updated',
        'content': 'New content',
    }
)
```

### การลบข้อมูล (Delete)

```python
# ลบ object เดียว
post = Post.objects.get(pk=1)
post.delete()  # ลบและคืน (จำนวน, {model: จำนวน})

# ลบหลาย objects
Post.objects.filter(status='archived').delete()

# ลบทั้งหมด (ระวัง!)
Post.objects.all().delete()
```

---

## 7. QuerySet API ขั้นสูง

### Ordering

```python
# เรียงลำดับ
Post.objects.order_by('created_at')        # ascending
Post.objects.order_by('-created_at')       # descending
Post.objects.order_by('category__name', '-created_at')  # หลาย fields

# สุ่ม
Post.objects.order_by('?')  # ไม่แนะนำสำหรับ production (ช้า)
```

### Slicing (Pagination)

```python
# ใช้ slice เหมือน Python list
posts = Post.objects.all()[:10]    # 10 อันแรก
posts = Post.objects.all()[10:20]  # อัน 11-20
first_post = Post.objects.all()[0]  # อันแรก

# first() และ last()
first = Post.objects.order_by('created_at').first()
last = Post.objects.order_by('created_at').last()
```

### Aggregation

```python
from django.db.models import Count, Avg, Sum, Max, Min, Q

# นับ posts ทั้งหมด
total = Post.objects.count()

# Aggregate functions
from django.db.models import Avg, Sum, Max, Min, Count

result = Post.objects.aggregate(
    total_posts=Count('id'),
    avg_views=Avg('views_count'),
    total_views=Sum('views_count'),
    max_views=Max('views_count'),
    min_views=Min('views_count'),
)
# {'total_posts': 50, 'avg_views': 234.5, ...}

# นับ posts ต่อหมวดหมู่
categories = Category.objects.annotate(
    post_count=Count('posts')
).order_by('-post_count')

for cat in categories:
    print(f'{cat.name}: {cat.post_count} posts')

# Annotate แบบซับซ้อน
from django.db.models import Count, Subquery, OuterRef

posts = Post.objects.annotate(
    comment_count=Count('comments')
).filter(comment_count__gt=5)
```

### Q Objects (Complex Queries)

```python
from django.db.models import Q

# OR condition
posts = Post.objects.filter(
    Q(status='published') | Q(is_featured=True)
)

# AND condition (ปกติใช้ filter kwargs แต่ Q ช่วยจัดกลุ่มได้)
posts = Post.objects.filter(
    Q(status='published') & Q(views_count__gt=100)
)

# NOT condition
posts = Post.objects.filter(
    ~Q(status='archived')
)

# ผสม Q objects
posts = Post.objects.filter(
    (Q(status='published') | Q(is_featured=True)) & ~Q(author__username='deleted_user')
)
```

### select_related และ prefetch_related

```python
# ปัญหา N+1 Query
posts = Post.objects.all()
for post in posts:
    print(post.author.username)  # query เพิ่มทุกครั้ง = N+1 queries!

# แก้ด้วย select_related (JOIN) - สำหรับ ForeignKey/OneToOne
posts = Post.objects.select_related('author', 'category').all()
for post in posts:
    print(post.author.username)  # ไม่มี extra query!

# prefetch_related - สำหรับ ManyToMany/reverse ForeignKey
posts = Post.objects.prefetch_related('tags', 'comments').all()
for post in posts:
    print([tag.name for tag in post.tags.all()])  # ไม่มี extra query!

# ผสมกัน
posts = Post.objects.select_related(
    'author', 'category'
).prefetch_related(
    'tags', 'comments__author'  # prefetch nested relations ได้
).all()
```

### only() และ defer()

```python
# ดึงเฉพาะ fields ที่ต้องการ (เร็วกว่าถ้า content ใหญ่)
posts = Post.objects.only('title', 'slug', 'created_at')

# ดึงทุก field ยกเว้น fields ที่ระบุ
posts = Post.objects.defer('content')  # ข้ามเนื้อหา
```

---

## 8. Custom Manager

```python
# blog/models.py

class PublishedManager(models.Manager):
    """Manager เฉพาะสำหรับ published posts"""
    def get_queryset(self):
        return super().get_queryset().filter(status=Post.Status.PUBLISHED)
    
    def featured(self):
        """ดึง posts ที่เป็น featured"""
        return self.get_queryset().filter(is_featured=True)
    
    def by_author(self, author):
        """ดึง posts ของ author คนนั้น"""
        return self.get_queryset().filter(author=author)


class Post(models.Model):
    # ...fields...
    
    # Default manager
    objects = models.Manager()
    
    # Custom manager
    published = PublishedManager()


# การใช้งาน
all_posts = Post.objects.all()              # ใช้ default manager
published = Post.published.all()           # ใช้ custom manager
featured = Post.published.featured()      # chain methods
```

---

## 9. Model Methods และ Properties

```python
class Post(models.Model):
    # ...
    
    def get_short_content(self, length=200):
        """ตัดเนื้อหาให้สั้น"""
        if len(self.content) > length:
            return self.content[:length] + '...'
        return self.content
    
    @property
    def word_count(self):
        """นับจำนวนคำ"""
        return len(self.content.split())
    
    @property
    def tags_list(self):
        """ดึง list ของชื่อ tag"""
        return list(self.tags.values_list('name', flat=True))
    
    @classmethod
    def get_trending(cls, limit=5):
        """ดึงบทความ trending"""
        return cls.published.order_by('-views_count')[:limit]
    
    @classmethod
    def get_recent(cls, days=7, limit=10):
        """ดึงบทความล่าสุดในช่วงเวลาที่กำหนด"""
        from datetime import timedelta
        since = timezone.now() - timedelta(days=days)
        return cls.published.filter(
            published_at__gte=since
        ).order_by('-published_at')[:limit]
```

---

## 10. Model Signals

```python
# blog/signals.py
from django.db.models.signals import post_save, pre_delete
from django.dispatch import receiver
from .models import Post


@receiver(post_save, sender=Post)
def post_saved(sender, instance, created, **kwargs):
    """
    ทำงานหลัง save Post
    created = True ถ้าเป็นการสร้างใหม่
    """
    if created:
        print(f'สร้างบทความใหม่: {instance.title}')
    else:
        print(f'แก้ไขบทความ: {instance.title}')
    
    # ตัวอย่าง: ส่ง notification
    # notify_followers(instance)


@receiver(pre_delete, sender=Post)
def post_deleting(sender, instance, **kwargs):
    """ทำงานก่อน delete Post"""
    print(f'กำลังลบบทความ: {instance.title}')
    # ลบไฟล์รูปภาพที่เกี่ยวข้องด้วย
    if instance.featured_image:
        instance.featured_image.delete(save=False)


# blog/apps.py
from django.apps import AppConfig

class BlogConfig(AppConfig):
    name = 'blog'
    verbose_name = 'บล็อก'
    
    def ready(self):
        """import signals เมื่อ app ready"""
        import blog.signals
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: E-commerce Models
สร้าง models สำหรับ e-commerce ที่มี:
- `Product` (ชื่อ, ราคา, stock, หมวดหมู่, รูปภาพ, คำอธิบาย)
- `Category` (ชื่อ, parent category สำหรับ nested)
- `Order` (user, status, total_amount, วันที่)
- `OrderItem` (order, product, quantity, price)

พร้อม:
- `__str__` method ทุก model
- Meta class ที่เหมาะสม
- Custom manager สำหรับ active products

### แบบฝึกหัดที่ 2: ORM Practice
ใช้ Django shell เพื่อ:
1. สร้าง categories 3 หมวดหมู่
2. สร้าง posts 5 บทความในแต่ละหมวดหมู่
3. Query: หาบทความที่มี views มากกว่า 100
4. Query: นับ posts ต่อหมวดหมู่
5. Query: หาบทความล่าสุด 3 อัน พร้อม author name (ไม่มี N+1)

### แบบฝึกหัดที่ 3: Model Methods
เพิ่ม method ให้ `Post` model:
- `get_related_posts()` - ดึงบทความที่มี category เดียวกัน (ยกเว้นตัวเอง) 3 อัน
- `get_next_post()` - ดึงบทความถัดไป (created_at มากกว่า)
- `get_previous_post()` - ดึงบทความก่อนหน้า

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- Django Model และ Field types ทุกประเภท
- Field options (null, blank, default, choices, unique)
- `__str__` method และ Meta class
- Django ORM สำหรับ CRUD operations
- QuerySet API (filter, exclude, annotate, aggregate)
- Q objects สำหรับ complex queries
- select_related และ prefetch_related แก้ปัญหา N+1
- Custom Manager และ Model Methods
- Model Signals

---

## บทถัดไป

➡️ **[Part 53: Django Migrations](part-053.md)** - เรียนรู้การจัดการ database schema ด้วย migrations
