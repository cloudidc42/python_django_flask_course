# Part 57: Django Admin

## เป้าหมายของบทเรียน

- ลงทะเบียน Models ใน Admin
- Customize ModelAdmin
- ใช้ list_display, list_filter, search_fields
- กำหนด fieldsets
- ใช้ Inline Admin
- สร้าง custom admin actions
- Customize Admin site

---

## 1. Django Admin คืออะไร?

Django Admin เป็น interface สำหรับจัดการข้อมูลในฐานข้อมูลที่สร้างอัตโนมัติจาก models

**คุณสมบัติ:**
- CRUD operations สำหรับทุก model
- ค้นหาและกรองข้อมูล
- จัดการ user permissions
- Customizable สูง

```bash
# สร้าง superuser
python manage.py createsuperuser

# เข้าสู่ admin
# http://127.0.0.1:8000/admin/
```

---

## 2. ลงทะเบียน Models

### วิธีพื้นฐาน

```python
# blog/admin.py
from django.contrib import admin
from .models import Category, Tag, Post, Comment

# วิธีที่ 1: ลงทะเบียนง่ายๆ
admin.site.register(Category)
admin.site.register(Tag)
```

### ใช้ @admin.register Decorator

```python
# blog/admin.py
from django.contrib import admin
from .models import Category, Tag, Post, Comment


@admin.register(Category)
class CategoryAdmin(admin.ModelAdmin):
    """Admin configuration สำหรับ Category model"""
    pass


@admin.register(Tag)
class TagAdmin(admin.ModelAdmin):
    pass
```

---

## 3. ModelAdmin Customization

### list_display

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    # columns ที่แสดงในหน้า list
    list_display = [
        'title',             # field โดยตรง
        'author',            # ForeignKey (แสดง __str__)
        'category',
        'status',
        'is_featured',
        'created_at',
        'views_count',
        'get_comment_count',  # custom method
    ]
    
    def get_comment_count(self, obj):
        """Column แสดงจำนวน comments"""
        return obj.comments.count()
    
    # กำหนด header ของ column
    get_comment_count.short_description = 'ความคิดเห็น'
    
    # ทำให้ column นี้ sortable ได้
    get_comment_count.admin_order_field = 'comments__count'
    
    # ทำให้ column แสดงสีหรือ HTML
    def colored_status(self, obj):
        from django.utils.html import format_html
        colors = {
            'draft': '#888',
            'published': '#28a745',
            'archived': '#dc3545',
        }
        color = colors.get(obj.status, '#000')
        return format_html(
            '<span style="color: {};">{}</span>',
            color,
            obj.get_status_display()
        )
    
    colored_status.short_description = 'สถานะ'
```

### list_filter

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    # sidebar filters
    list_filter = [
        'status',           # field โดยตรง
        'is_featured',
        'created_at',       # DateField สร้าง date hierarchy filter
        'author',           # ForeignKey
        'category',
        'tags',             # ManyToMany
    ]
    
    # Custom filter
    # (ดูด้านล่าง)
```

### search_fields

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    # fields ที่ค้นหาได้
    search_fields = [
        'title',            # ค้นหาใน title
        'content',          # ค้นหาใน content
        'author__username', # ค้นหา related field
        'author__email',
        'tags__name',
    ]
    
    # ใน search: django จะทำ OR query ระหว่าง fields ทั้งหมด
```

### date_hierarchy

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    # navigation ด้วย date
    date_hierarchy = 'created_at'
```

### ordering

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    # การเรียงลำดับ default ในหน้า list
    ordering = ['-created_at']
```

### list_per_page และ list_max_show_all

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_per_page = 25         # จำนวนต่อหน้า (default: 100)
    list_max_show_all = 1000   # จำนวนสูงสุดที่ show all ได้
```

### list_editable

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ['title', 'status', 'is_featured']
    # fields ที่แก้ไขได้โดยตรงในหน้า list (ต้องอยู่ใน list_display ด้วย)
    list_editable = ['status', 'is_featured']
```

---

## 4. fieldsets

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    # จัดกลุ่ม fields ในหน้า detail
    fieldsets = [
        # (ชื่อกลุ่ม, {options})
        (None, {  # None = ไม่มีชื่อ
            'fields': ['title', 'slug', 'author']
        }),
        ('เนื้อหา', {
            'fields': ['excerpt', 'content', 'featured_image'],
            'description': 'กรอกเนื้อหาบทความ',
        }),
        ('การจัดหมวดหมู่', {
            'fields': ['category', 'tags'],
        }),
        ('การเผยแพร่', {
            'fields': ['status', 'published_at', 'is_featured'],
        }),
        ('ข้อมูลเพิ่มเติม', {
            'fields': ['views_count'],
            'classes': ['collapse'],  # ซ่อน/แสดงได้
            'description': 'ข้อมูลสถิติ',
        }),
    ]
    
    # prepopulated fields - กรอกอัตโนมัติ
    prepopulated_fields = {'slug': ('title',)}
    
    # fields แบบ readonly
    readonly_fields = ['created_at', 'updated_at', 'views_count']
    
    # raw_id_fields สำหรับ ForeignKey ที่มีข้อมูลเยอะ
    raw_id_fields = ['author']
    
    # autocomplete_fields (ต้องมี search_fields ใน related admin)
    # autocomplete_fields = ['category']
    
    # filter_horizontal สำหรับ ManyToMany
    filter_horizontal = ['tags']
    # หรือ filter_vertical
    # filter_vertical = ['tags']
```

---

## 5. Inline Admin

Inlines ให้แก้ไข related objects ในหน้าเดียวกัน

### TabularInline

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    # ...
    pass


class CommentInline(admin.TabularInline):
    """แสดง comments ใน Post admin"""
    model = Comment
    extra = 0              # ไม่แสดง empty forms
    max_num = 10           # จำนวน inline สูงสุด
    readonly_fields = ['author', 'created_at']
    fields = ['author', 'content', 'is_approved', 'created_at']
    show_change_link = True  # แสดง link ไปยัง comment detail
    
    # กรอง inline ตาม condition
    def get_queryset(self, request):
        return super().get_queryset(request).select_related('author')


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    # เพิ่ม inline
    inlines = [CommentInline]
    
    fieldsets = [
        (None, {'fields': ['title', 'slug', 'author', 'content', 'status']}),
    ]
```

### StackedInline

```python
class OrderItemInline(admin.StackedInline):
    """แสดง items ใน Order admin"""
    model = OrderItem
    extra = 0
    readonly_fields = ['product', 'price']
    fields = ['product', 'quantity', 'price']


@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    inlines = [OrderItemInline]
    list_display = ['id', 'user', 'status', 'total_amount', 'created_at']
```

---

## 6. Custom Admin Actions

```python
from django.contrib import admin, messages
from django.utils import timezone


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ['title', 'status', 'author', 'created_at']
    list_filter = ['status']
    actions = ['publish_posts', 'archive_posts', 'send_newsletter']
    
    def publish_posts(self, request, queryset):
        """
        Action สำหรับเผยแพร่บทความที่เลือก
        """
        # queryset คือ objects ที่ถูกเลือก
        updated = queryset.update(
            status='published',
            published_at=timezone.now()
        )
        self.message_user(
            request,
            f'เผยแพร่ {updated} บทความสำเร็จ',
            messages.SUCCESS
        )
    
    # กำหนดชื่อที่แสดงใน dropdown
    publish_posts.short_description = 'เผยแพร่บทความที่เลือก'
    
    def archive_posts(self, request, queryset):
        """Action สำหรับเก็บบทความ"""
        updated = queryset.update(status='archived')
        self.message_user(
            request,
            f'เก็บ {updated} บทความสำเร็จ',
            messages.SUCCESS
        )
    
    archive_posts.short_description = 'เก็บบทความที่เลือก'
    
    def send_newsletter(self, request, queryset):
        """Action สำหรับส่ง newsletter"""
        published_posts = queryset.filter(status='published')
        if not published_posts.exists():
            self.message_user(
                request,
                'ไม่มีบทความที่เผยแพร่แล้ว',
                messages.WARNING
            )
            return
        
        # TODO: ส่ง email จริงๆ
        count = published_posts.count()
        self.message_user(
            request,
            f'ส่ง newsletter สำหรับ {count} บทความสำเร็จ',
            messages.SUCCESS
        )
    
    send_newsletter.short_description = 'ส่ง Newsletter'
    
    # กำหนด permission สำหรับ action
    def send_newsletter(self, request, queryset):
        pass
    
    send_newsletter.allowed_permissions = ('change',)  # ต้องมี change permission
```

### Custom Action กับ Intermediate Form

```python
from django.shortcuts import render
from django.http import HttpResponseRedirect


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    actions = ['export_posts']
    
    def export_posts(self, request, queryset):
        """
        Action ที่ redirect ไปยังหน้า form ก่อน
        """
        # ถ้ากด Confirm
        if 'apply' in request.POST:
            export_format = request.POST.get('format', 'csv')
            
            # ทำการ export
            if export_format == 'csv':
                return self._export_csv(queryset)
            else:
                return self._export_json(queryset)
        
        # แสดง intermediate form
        return render(
            request,
            'admin/blog/post/export_form.html',
            {
                'posts': queryset,
                'action_checkbox_name': admin.helpers.ACTION_CHECKBOX_NAME,
            }
        )
    
    export_posts.short_description = 'Export บทความที่เลือก'
    
    def _export_csv(self, queryset):
        import csv
        from django.http import HttpResponse
        
        response = HttpResponse(content_type='text/csv; charset=utf-8')
        response['Content-Disposition'] = 'attachment; filename="posts.csv"'
        response.write('﻿')  # UTF-8 BOM สำหรับ Excel
        
        writer = csv.writer(response)
        writer.writerow(['ID', 'Title', 'Author', 'Status', 'Created At'])
        
        for post in queryset:
            writer.writerow([
                post.pk,
                post.title,
                post.author.username,
                post.status,
                post.created_at.strftime('%Y-%m-%d'),
            ])
        
        return response
```

---

## 7. Custom Admin Methods

```python
from django.utils.html import format_html
from django.utils.safestring import mark_safe


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = [
        'title',
        'author_link',
        'status_badge',
        'featured_image_thumbnail',
        'post_url',
    ]
    
    def author_link(self, obj):
        """แสดงชื่อ author พร้อม link ไปยัง user admin"""
        from django.urls import reverse
        url = reverse('admin:auth_user_change', args=[obj.author.pk])
        return format_html('<a href="{}">{}</a>', url, obj.author.username)
    
    author_link.short_description = 'ผู้เขียน'
    author_link.admin_order_field = 'author__username'
    
    def status_badge(self, obj):
        """แสดง status เป็น badge สี"""
        colors = {
            'draft': 'secondary',
            'published': 'success',
            'archived': 'danger',
        }
        color = colors.get(obj.status, 'secondary')
        return format_html(
            '<span class="badge text-bg-{}">{}</span>',
            color,
            obj.get_status_display()
        )
    
    status_badge.short_description = 'สถานะ'
    
    def featured_image_thumbnail(self, obj):
        """แสดง thumbnail ของรูปภาพ"""
        if obj.featured_image:
            return format_html(
                '<img src="{}" style="max-height: 50px; max-width: 100px;" />',
                obj.featured_image.url
            )
        return '-'
    
    featured_image_thumbnail.short_description = 'รูปภาพ'
    
    def post_url(self, obj):
        """แสดง link ไปยังหน้า public"""
        if obj.status == 'published':
            url = obj.get_absolute_url()
            return format_html('<a href="{}" target="_blank">ดูบทความ →</a>', url)
        return '-'
    
    post_url.short_description = 'ลิงก์'
```

---

## 8. Custom Filters

```python
from django.contrib.admin import SimpleListFilter
from django.utils import timezone
from datetime import timedelta


class PublishedThisWeekFilter(SimpleListFilter):
    """
    Filter: เผยแพร่ในสัปดาห์นี้
    """
    title = 'เผยแพร่เมื่อ'  # ชื่อที่แสดงใน sidebar
    parameter_name = 'published_when'  # URL parameter name
    
    def lookups(self, request, model_admin):
        """
        คืน list ของ (value, display_name) สำหรับ filter options
        """
        return [
            ('today', 'วันนี้'),
            ('week', 'สัปดาห์นี้'),
            ('month', 'เดือนนี้'),
        ]
    
    def queryset(self, request, queryset):
        """
        Apply filter ตาม value ที่เลือก
        """
        now = timezone.now()
        
        if self.value() == 'today':
            return queryset.filter(
                published_at__date=now.date()
            )
        elif self.value() == 'week':
            week_ago = now - timedelta(days=7)
            return queryset.filter(
                published_at__gte=week_ago
            )
        elif self.value() == 'month':
            month_ago = now - timedelta(days=30)
            return queryset.filter(
                published_at__gte=month_ago
            )
        return queryset


class HasFeaturedImageFilter(SimpleListFilter):
    """Filter: มีรูปภาพหรือไม่"""
    title = 'รูปภาพ'
    parameter_name = 'has_image'
    
    def lookups(self, request, model_admin):
        return [
            ('yes', 'มีรูปภาพ'),
            ('no', 'ไม่มีรูปภาพ'),
        ]
    
    def queryset(self, request, queryset):
        if self.value() == 'yes':
            return queryset.exclude(featured_image='')
        elif self.value() == 'no':
            return queryset.filter(featured_image='')
        return queryset


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_filter = [
        'status',
        'is_featured',
        PublishedThisWeekFilter,  # custom filter
        HasFeaturedImageFilter,
    ]
```

---

## 9. Admin สำหรับ Blog แบบสมบูรณ์

```python
# blog/admin.py
from django.contrib import admin
from django.utils.html import format_html
from django.utils import timezone
from django.urls import reverse
from .models import Category, Tag, Post, Comment


@admin.register(Category)
class CategoryAdmin(admin.ModelAdmin):
    list_display = ['name', 'slug', 'color_preview', 'post_count', 'created_at']
    search_fields = ['name', 'description']
    prepopulated_fields = {'slug': ('name',)}
    readonly_fields = ['created_at']
    
    def color_preview(self, obj):
        """แสดงสีตัวอย่าง"""
        return format_html(
            '<span style="background-color: {}; display: inline-block; '
            'width: 20px; height: 20px; border-radius: 50%; '
            'border: 1px solid #ccc;"></span> {}',
            obj.color, obj.color
        )
    color_preview.short_description = 'สี'
    
    def post_count(self, obj):
        count = obj.posts.count()
        url = reverse('admin:blog_post_changelist') + f'?category__id={obj.pk}'
        return format_html('<a href="{}">{} บทความ</a>', url, count)
    post_count.short_description = 'จำนวนบทความ'


@admin.register(Tag)
class TagAdmin(admin.ModelAdmin):
    list_display = ['name', 'slug', 'post_count']
    search_fields = ['name']
    prepopulated_fields = {'slug': ('name',)}
    
    def post_count(self, obj):
        return obj.posts.count()
    post_count.short_description = 'จำนวนบทความ'


class CommentInline(admin.TabularInline):
    model = Comment
    extra = 0
    readonly_fields = ['author', 'created_at']
    fields = ['author', 'content', 'is_approved', 'created_at']
    can_delete = True
    show_change_link = True
    
    def get_queryset(self, request):
        return super().get_queryset(request).select_related('author').filter(
            parent__isnull=True  # แสดงเฉพาะ top-level comments
        )


@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = [
        'title',
        'author_link',
        'category',
        'status_badge',
        'is_featured',
        'thumbnail',
        'views_count',
        'comment_count',
        'created_at',
    ]
    list_filter = [
        'status',
        'is_featured',
        'category',
        'created_at',
    ]
    search_fields = ['title', 'content', 'author__username', 'tags__name']
    date_hierarchy = 'created_at'
    ordering = ['-created_at']
    list_per_page = 20
    list_editable = ['is_featured']
    actions = ['publish_posts', 'archive_posts', 'export_csv']
    
    fieldsets = [
        ('ข้อมูลพื้นฐาน', {
            'fields': ['title', 'slug', 'author', 'featured_image'],
        }),
        ('เนื้อหา', {
            'fields': ['excerpt', 'content'],
        }),
        ('การจัดหมวดหมู่', {
            'fields': ['category', 'tags'],
        }),
        ('การเผยแพร่', {
            'fields': ['status', 'is_featured', 'published_at'],
        }),
        ('สถิติ', {
            'fields': ['views_count', 'created_at', 'updated_at'],
            'classes': ['collapse'],
        }),
    ]
    
    prepopulated_fields = {'slug': ('title',)}
    readonly_fields = ['views_count', 'created_at', 'updated_at']
    filter_horizontal = ['tags']
    raw_id_fields = ['author']
    
    inlines = [CommentInline]
    
    # === Custom columns ===
    
    def author_link(self, obj):
        url = reverse('admin:auth_user_change', args=[obj.author.pk])
        return format_html('<a href="{}">{}</a>', url, obj.author.get_full_name() or obj.author.username)
    author_link.short_description = 'ผู้เขียน'
    author_link.admin_order_field = 'author__username'
    
    def status_badge(self, obj):
        colors = {'draft': '#888', 'published': '#28a745', 'archived': '#dc3545'}
        color = colors.get(obj.status, '#888')
        return format_html(
            '<span style="color: white; background-color: {}; padding: 2px 8px; '
            'border-radius: 3px; font-size: 11px;">{}</span>',
            color, obj.get_status_display()
        )
    status_badge.short_description = 'สถานะ'
    
    def thumbnail(self, obj):
        if obj.featured_image:
            return format_html(
                '<img src="{}" style="height: 40px; width: 60px; object-fit: cover; border-radius: 3px;" />',
                obj.featured_image.url
            )
        return format_html('<span style="color: #ccc;">ไม่มีรูป</span>')
    thumbnail.short_description = 'รูป'
    
    def comment_count(self, obj):
        count = obj.comments.filter(is_approved=True).count()
        url = reverse('admin:blog_comment_changelist') + f'?post__id={obj.pk}'
        return format_html('<a href="{}">{}</a>', url, count)
    comment_count.short_description = 'ความคิดเห็น'
    
    # === Actions ===
    
    def publish_posts(self, request, queryset):
        updated = queryset.filter(status='draft').update(
            status='published',
            published_at=timezone.now()
        )
        self.message_user(request, f'เผยแพร่ {updated} บทความสำเร็จ')
    publish_posts.short_description = 'เผยแพร่บทความที่เลือก'
    
    def archive_posts(self, request, queryset):
        updated = queryset.update(status='archived')
        self.message_user(request, f'เก็บ {updated} บทความสำเร็จ')
    archive_posts.short_description = 'เก็บบทความที่เลือก'
    
    def export_csv(self, request, queryset):
        import csv
        from django.http import HttpResponse
        
        response = HttpResponse(content_type='text/csv; charset=utf-8-sig')
        response['Content-Disposition'] = 'attachment; filename="posts.csv"'
        
        writer = csv.writer(response)
        writer.writerow(['ID', 'หัวข้อ', 'ผู้เขียน', 'หมวดหมู่', 'สถานะ', 'วันที่สร้าง'])
        
        for post in queryset:
            writer.writerow([
                post.pk,
                post.title,
                post.author.username,
                post.category.name if post.category else '',
                post.get_status_display(),
                post.created_at.strftime('%Y-%m-%d'),
            ])
        
        return response
    export_csv.short_description = 'Export เป็น CSV'
    
    # === Queryset optimization ===
    
    def get_queryset(self, request):
        return super().get_queryset(request).select_related(
            'author', 'category'
        ).prefetch_related('tags', 'comments')


@admin.register(Comment)
class CommentAdmin(admin.ModelAdmin):
    list_display = ['short_content', 'author', 'post_link', 'is_approved', 'created_at']
    list_filter = ['is_approved', 'created_at']
    search_fields = ['content', 'author__username', 'post__title']
    list_editable = ['is_approved']
    readonly_fields = ['author', 'post', 'created_at']
    actions = ['approve_comments', 'reject_comments']
    
    def short_content(self, obj):
        return obj.content[:50] + '...' if len(obj.content) > 50 else obj.content
    short_content.short_description = 'ความคิดเห็น'
    
    def post_link(self, obj):
        url = reverse('admin:blog_post_change', args=[obj.post.pk])
        return format_html('<a href="{}">{}</a>', url, obj.post.title[:30])
    post_link.short_description = 'บทความ'
    
    def approve_comments(self, request, queryset):
        queryset.update(is_approved=True)
        self.message_user(request, f'อนุมัติ {queryset.count()} ความคิดเห็นสำเร็จ')
    approve_comments.short_description = 'อนุมัติความคิดเห็นที่เลือก'
    
    def reject_comments(self, request, queryset):
        queryset.update(is_approved=False)
        self.message_user(request, f'ปฏิเสธ {queryset.count()} ความคิดเห็น')
    reject_comments.short_description = 'ปฏิเสธความคิดเห็นที่เลือก'
    
    def get_queryset(self, request):
        return super().get_queryset(request).select_related('author', 'post')
```

---

## 10. Customize Admin Site

```python
# mysite/admin.py หรือ ใน apps.py
from django.contrib import admin

# เปลี่ยนชื่อ admin site
admin.site.site_header = 'MyBlog Admin'
admin.site.site_title = 'MyBlog Admin Portal'
admin.site.index_title = 'ยินดีต้อนรับสู่ MyBlog Admin'
```

### Custom Admin Template

```bash
# สร้าง templates/admin/base_site.html
```

```html
<!-- templates/admin/base_site.html -->
{% extends "admin/base.html" %}
{% load static %}

{% block branding %}
<h1 id="site-name">
    <a href="{% url 'admin:index' %}">
        <img src="{% static 'images/logo.png' %}" height="30" alt="Logo">
        MyBlog Admin
    </a>
</h1>
{% endblock %}

{% block extrastyle %}
{{ block.super }}
<style>
    /* Custom admin styles */
    #header {
        background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    }
    .module h2, .module caption, .inline-group h2 {
        background: #667eea;
    }
</style>
{% endblock %}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: E-commerce Admin
สร้าง Admin สำหรับ e-commerce ที่มี:
- ProductAdmin: list_display, search, filter, thumbnail
- OrderAdmin: แสดง items inline, status filter, total amount
- Custom action: mark orders as shipped

### แบบฝึกหัดที่ 2: Custom Filter
สร้าง custom filter:
- `PriceRangeFilter` สำหรับ Product: ต่ำกว่า 100, 100-500, สูงกว่า 500
- `LowStockFilter` สำหรับ Product: stock < 10

### แบบฝึกหัดที่ 3: Export Action
สร้าง export action ที่:
- Export เป็น CSV พร้อม BOM สำหรับ Excel ภาษาไทย
- Export เป็น Excel (.xlsx) ด้วย openpyxl
- แสดง intermediate form เพื่อเลือก format

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- ลงทะเบียน Models ใน Admin
- list_display, list_filter, search_fields
- fieldsets และ prepopulated_fields
- Inline Admin (Tabular, Stacked)
- Custom actions
- Custom list columns ด้วย format_html
- Custom filters ด้วย SimpleListFilter
- Customize admin site appearance

---

## บทถัดไป

➡️ **[Part 58: Django Forms](part-058.md)** - เรียนรู้การสร้างและจัดการ Forms ใน Django
