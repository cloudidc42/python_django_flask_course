# Part 55: Django Templates

## เป้าหมายของบทเรียน

- เข้าใจ Django Template Language (DTL)
- ใช้ Template tags และ Template filters
- สร้าง Template inheritance
- จัดการ Static files
- Integration กับ Bootstrap 5
- สร้าง custom template tags และ filters

---

## 1. Django Template Language (DTL)

Django มี template language พิเศษที่ออกแบบมาสำหรับ web development

**ประเภทของ syntax:**
- `{{ variable }}` - แสดงค่าตัวแปร
- `{% tag %}` - template tag (logic)
- `{# comment #}` - comment

---

## 2. Variables และ Lookups

```html
<!-- แสดงค่าตัวแปรจาก context -->
<h1>{{ title }}</h1>
<p>{{ post.content }}</p>

<!-- Attribute lookup -->
{{ post.author.username }}

<!-- Dictionary lookup -->
{{ user_info.email }}

<!-- List index -->
{{ items.0 }}  <!-- items[0] -->

<!-- ถ้าตัวแปรไม่มีค่า Django แสดง '' (empty string) -->
<!-- หรือใช้ default filter -->
{{ name|default:"ไม่ระบุ" }}
```

---

## 3. Template Tags

### if / elif / else

```html
{% if user.is_authenticated %}
    <p>สวัสดี, {{ user.username }}!</p>
{% elif user.is_staff %}
    <p>Admin: {{ user.username }}</p>
{% else %}
    <p>กรุณา <a href="/login/">เข้าสู่ระบบ</a></p>
{% endif %}

<!-- เงื่อนไขแบบต่างๆ -->
{% if posts %}
    มีบทความ
{% endif %}

{% if count > 5 %}
    มากกว่า 5
{% endif %}

{% if status == "published" %}
    เผยแพร่แล้ว
{% endif %}

{% if user.is_authenticated and post.author == user %}
    เจ้าของบทความ
{% endif %}

{% if not post.is_published %}
    ยังไม่เผยแพร่
{% endif %}

{% if tag in post.tags.all %}
    มี tag นี้
{% endif %}
```

### for loop

```html
<!-- วนลูปแสดง posts -->
{% for post in posts %}
    <div class="post-card">
        <h2>{{ post.title }}</h2>
        <p>{{ post.excerpt }}</p>
        <a href="{{ post.get_absolute_url }}">อ่านต่อ</a>
    </div>
{% empty %}
    <!-- แสดงเมื่อ list ว่าง -->
    <p>ยังไม่มีบทความ</p>
{% endfor %}

<!-- Loop variables ที่มีให้ใช้ -->
{% for item in items %}
    <tr>
        <td>{{ forloop.counter }}</td>      <!-- 1, 2, 3, ... -->
        <td>{{ forloop.counter0 }}</td>     <!-- 0, 1, 2, ... -->
        <td>{{ forloop.revcounter }}</td>   <!-- N, N-1, ..., 1 -->
        <td>{{ forloop.revcounter0 }}</td>  <!-- N-1, ..., 0 -->
        <td>{{ forloop.first }}</td>        <!-- True ถ้าเป็นตัวแรก -->
        <td>{{ forloop.last }}</td>         <!-- True ถ้าเป็นตัวสุดท้าย -->
        <td>{{ forloop.parentloop.counter }}</td>  <!-- outer loop counter -->
        <td>{{ item.name }}</td>
    </tr>
{% endfor %}

<!-- วนลูปแบบ nested -->
{% for category in categories %}
    <h3>{{ category.name }}</h3>
    <ul>
        {% for post in category.posts.all %}
            <li>{{ post.title }}</li>
        {% endfor %}
    </ul>
{% endfor %}
```

### block และ extends (Template Inheritance)

```html
<!-- base.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    
    {% load static %}
    
    <!-- Block สำหรับ title -->
    <title>{% block title %}บล็อกของเรา{% endblock %}</title>
    
    <!-- CSS ที่ใช้ทุกหน้า -->
    <link rel="stylesheet" href="{% static 'css/main.css' %}">
    
    <!-- Block สำหรับ extra CSS -->
    {% block extra_css %}{% endblock %}
</head>
<body>
    <!-- Navigation -->
    <nav>
        <a href="{% url 'blog:post_list' %}">หน้าแรก</a>
        <a href="{% url 'pages:about' %}">เกี่ยวกับเรา</a>
        {% if user.is_authenticated %}
            <a href="{% url 'blog:post_create' %}">เขียนบทความ</a>
            <a href="{% url 'logout' %}">ออกจากระบบ</a>
        {% else %}
            <a href="{% url 'login' %}">เข้าสู่ระบบ</a>
        {% endif %}
    </nav>
    
    <!-- Messages -->
    {% if messages %}
    <div class="messages">
        {% for message in messages %}
        <div class="alert alert-{{ message.tags }}">
            {{ message }}
        </div>
        {% endfor %}
    </div>
    {% endif %}
    
    <!-- Main content block -->
    <main>
        {% block content %}
        <!-- หน้า child จะ override block นี้ -->
        {% endblock %}
    </main>
    
    <!-- Footer -->
    <footer>
        <p>&copy; 2026 บล็อกของเรา</p>
    </footer>
    
    <!-- JS ที่ใช้ทุกหน้า -->
    <script src="{% static 'js/main.js' %}"></script>
    
    <!-- Block สำหรับ extra JS -->
    {% block extra_js %}{% endblock %}
</body>
</html>
```

```html
<!-- blog/post_list.html - extends base -->
{% extends 'base.html' %}
{% load static %}

<!-- Override title block -->
{% block title %}รายการบทความ | {{ block.super }}{% endblock %}
<!-- block.super = เนื้อหาเดิมของ parent block -->

{% block extra_css %}
<style>
    .post-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 20px; }
</style>
{% endblock %}

{% block content %}
<h1>บทความทั้งหมด</h1>

<!-- Search form -->
<form method="get">
    <input type="text" name="q" value="{{ search_query }}" placeholder="ค้นหา...">
    <button type="submit">ค้นหา</button>
</form>

<!-- Post grid -->
<div class="post-grid">
    {% for post in posts %}
    <article class="post-card">
        {% if post.featured_image %}
        <img src="{{ post.featured_image.url }}" alt="{{ post.title }}">
        {% else %}
        <img src="{% static 'images/default-post.jpg' %}" alt="{{ post.title }}">
        {% endif %}
        
        <h2><a href="{{ post.get_absolute_url }}">{{ post.title }}</a></h2>
        
        <div class="meta">
            <span>{{ post.author.get_full_name|default:post.author.username }}</span>
            <span>{{ post.created_at|date:"d M Y" }}</span>
            <span>{{ post.reading_time }} นาทีในการอ่าน</span>
        </div>
        
        {% if post.excerpt %}
        <p>{{ post.excerpt|truncatewords:30 }}</p>
        {% else %}
        <p>{{ post.content|truncatewords:30 }}</p>
        {% endif %}
        
        <!-- Tags -->
        <div class="tags">
            {% for tag in post.tags.all %}
            <span class="tag">#{{ tag.name }}</span>
            {% endfor %}
        </div>
        
        <a href="{{ post.get_absolute_url }}" class="btn">อ่านต่อ</a>
    </article>
    {% empty %}
    <p>ยังไม่มีบทความ</p>
    {% endfor %}
</div>

<!-- Pagination -->
{% if page_obj.has_other_pages %}
<div class="pagination">
    {% if page_obj.has_previous %}
    <a href="?page={{ page_obj.previous_page_number }}{% if search_query %}&q={{ search_query }}{% endif %}">
        &laquo; ก่อนหน้า
    </a>
    {% endif %}
    
    {% for num in page_obj.paginator.page_range %}
        {% if page_obj.number == num %}
        <span class="current">{{ num }}</span>
        {% elif num > page_obj.number|add:'-3' and num < page_obj.number|add:'3' %}
        <a href="?page={{ num }}">{{ num }}</a>
        {% endif %}
    {% endfor %}
    
    {% if page_obj.has_next %}
    <a href="?page={{ page_obj.next_page_number }}{% if search_query %}&q={{ search_query }}{% endif %}">
        ถัดไป &raquo;
    </a>
    {% endif %}
</div>
{% endif %}

{% endblock %}
```

### include

```html
<!-- blog/post_detail.html -->
{% extends 'base.html' %}

{% block content %}
<article>
    <h1>{{ post.title }}</h1>
    <div class="post-content">{{ post.content|linebreaks }}</div>
</article>

<!-- Include partial template -->
{% include 'blog/partials/comment_list.html' with comments=comments %}
{% include 'blog/partials/comment_form.html' with form=comment_form %}
{% include 'blog/partials/related_posts.html' with posts=related_posts %}

{% endblock %}
```

```html
<!-- blog/partials/comment_list.html -->
<section class="comments">
    <h3>ความคิดเห็น ({{ comments|length }})</h3>
    
    {% for comment in comments %}
    <div class="comment" id="comment-{{ comment.pk }}">
        <div class="comment-header">
            <strong>{{ comment.author.username }}</strong>
            <span>{{ comment.created_at|timesince }} ที่แล้ว</span>
        </div>
        <div class="comment-body">{{ comment.content|linebreaks }}</div>
        
        <!-- Replies -->
        {% if comment.replies.all %}
        <div class="replies">
            {% for reply in comment.replies.all %}
            {% include 'blog/partials/comment_single.html' with comment=reply %}
            {% endfor %}
        </div>
        {% endif %}
    </div>
    {% empty %}
    <p>ยังไม่มีความคิดเห็น เป็นคนแรกที่แสดงความคิดเห็น!</p>
    {% endfor %}
</section>
```

### url tag

```html
<!-- URL โดยไม่มี parameter -->
<a href="{% url 'blog:post_list' %}">รายการบทความ</a>

<!-- URL กับ positional argument -->
<a href="{% url 'blog:post_detail' post.slug %}">{{ post.title }}</a>

<!-- URL กับ keyword argument -->
<a href="{% url 'blog:post_detail' slug=post.slug %}">{{ post.title }}</a>

<!-- URL กับหลาย parameters -->
<a href="{% url 'shop:product_detail' category_slug=category.slug product_slug=product.slug %}">
    {{ product.name }}
</a>

<!-- เก็บ URL ในตัวแปร -->
{% url 'blog:post_detail' slug=post.slug as post_url %}
<a href="{{ post_url }}">{{ post.title }}</a>
```

### with tag (variable aliasing)

```html
<!-- กำหนด alias เพื่อลด repeated lookups -->
{% with author=post.author %}
    <p>{{ author.username }}</p>
    <p>{{ author.email }}</p>
    <p>{{ author.get_full_name }}</p>
{% endwith %}

<!-- หลาย aliases -->
{% with author=post.author category=post.category %}
    <p>{{ author.username }} | {{ category.name }}</p>
{% endwith %}
```

### Other Built-in Tags

```html
<!-- csrf_token - จำเป็นสำหรับ POST forms -->
<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">บันทึก</button>
</form>

<!-- load - โหลด template tag library -->
{% load static %}
{% load i18n %}

<!-- comment - multi-line comment -->
{% comment "Optional note" %}
    เนื้อหาที่ไม่แสดง...
    อาจมีหลายบรรทัด
{% endcomment %}

<!-- cycle - สลับค่า -->
{% for item in items %}
<tr class="{% cycle 'odd' 'even' %}">
    <td>{{ item }}</td>
</tr>
{% endfor %}

<!-- firstof - แสดง value แรกที่ไม่ใช่ False -->
{% firstof var1 var2 var3 "default" %}

<!-- regroup - จัดกลุ่ม list -->
{% regroup posts by category as category_list %}
{% for category in category_list %}
    <h3>{{ category.grouper }}</h3>
    {% for post in category.list %}
        <p>{{ post.title }}</p>
    {% endfor %}
{% endfor %}
```

---

## 4. Template Filters

```html
<!-- String filters -->
{{ name|lower }}              <!-- ตัวพิมพ์เล็กทั้งหมด -->
{{ name|upper }}              <!-- ตัวพิมพ์ใหญ่ทั้งหมด -->
{{ name|title }}              <!-- ตัวพิมพ์ใหญ่คำแรกของแต่ละคำ -->
{{ name|capfirst }}           <!-- ตัวพิมพ์ใหญ่ตัวแรก -->
{{ name|swapcase }}           <!-- สลับ case -->

{{ name|length }}             <!-- ความยาว -->
{{ name|wordcount }}          <!-- จำนวนคำ -->

{{ text|truncatechars:100 }}  <!-- ตัดที่ 100 ตัวอักษร -->
{{ text|truncatewords:20 }}   <!-- ตัดที่ 20 คำ -->
{{ text|truncatewords_html:20 }} <!-- ตัดที่ 20 คำ (safe สำหรับ HTML) -->

{{ html_content|safe }}        <!-- mark HTML ว่าปลอดภัย (ระวัง XSS!) -->
{{ text|linebreaks }}          <!-- แปลง \n เป็น <p> -->
{{ text|linebreaksbr }}        <!-- แปลง \n เป็น <br> -->

{{ html|striptags }}           <!-- ลบ HTML tags -->
{{ html|escape }}              <!-- escape HTML entities -->

{{ text|urlize }}              <!-- แปลง URL เป็น links -->
{{ text|urlizetrunc:30 }}      <!-- urlize แต่ตัดข้อความ -->

<!-- Number filters -->
{{ price|floatformat:2 }}     <!-- แสดงทศนิยม 2 ตำแหน่ง: 1234.50 -->
{{ price|floatformat:0 }}     <!-- ไม่มีทศนิยม: 1235 -->
{{ count|intcomma }}           <!-- ใส่ comma: 1,234,567 -->
{{ bytes|filesizeformat }}     <!-- แปลง bytes: 1.2 MB -->

<!-- Date/Time filters -->
{{ post.created_at|date:"d/m/Y" }}     <!-- 02/10/2026 -->
{{ post.created_at|date:"d M Y H:i" }} <!-- 02 Oct 2026 10:30 -->
{{ post.created_at|date:"j F Y" }}     <!-- 2 October 2026 -->
{{ post.created_at|time:"H:i" }}       <!-- 10:30 -->
{{ post.created_at|timesince }}        <!-- "3 hours, 4 minutes" -->
{{ post.created_at|timeuntil }}        <!-- "2 days, 3 hours" -->

<!-- List/QuerySet filters -->
{{ items|length }}             <!-- จำนวน -->
{{ items|first }}              <!-- ค่าแรก -->
{{ items|last }}               <!-- ค่าสุดท้าย -->
{{ items|random }}             <!-- สุ่ม -->
{{ items|join:", " }}          <!-- เชื่อมด้วย separator: "a, b, c" -->
{{ numbers|add:5 }}            <!-- บวก 5 -->

<!-- Chaining filters -->
{{ post.title|lower|truncatechars:50 }}
{{ post.created_at|date:"d M Y"|lower }}

<!-- Default filter -->
{{ user.bio|default:"ยังไม่ได้กรอกข้อมูล" }}
{{ value|default_if_none:"N/A" }}
```

---

## 5. Static Files

### Configuration

```python
# settings.py

# URL prefix สำหรับ static files
STATIC_URL = '/static/'

# โฟลเดอร์ static files (development)
STATICFILES_DIRS = [
    BASE_DIR / 'static',
]

# โฟลเดอร์ที่ collectstatic จะรวมไฟล์ (production)
STATIC_ROOT = BASE_DIR / 'staticfiles'

# สำหรับ media files (user uploads)
MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'
```

### โครงสร้างโฟลเดอร์

```
static/
├── css/
│   ├── main.css
│   └── blog.css
├── js/
│   ├── main.js
│   └── blog.js
└── images/
    ├── logo.png
    └── default-post.jpg

# หรือในแต่ละ app
blog/
└── static/
    └── blog/
        ├── css/
        │   └── blog.css
        └── js/
            └── blog.js
```

### ใช้ Static Files ใน Template

```html
{% load static %}

<!-- CSS -->
<link rel="stylesheet" href="{% static 'css/main.css' %}">
<link rel="stylesheet" href="{% static 'blog/css/blog.css' %}">

<!-- JavaScript -->
<script src="{% static 'js/main.js' %}"></script>

<!-- Images -->
<img src="{% static 'images/logo.png' %}" alt="Logo">

<!-- เก็บ URL ในตัวแปร -->
{% static 'images/default.jpg' as default_img %}
<img src="{{ default_img }}" alt="Default">

<!-- Media files (user uploads) -->
{% if post.featured_image %}
<img src="{{ post.featured_image.url }}" alt="{{ post.title }}">
{% endif %}
```

### Serve Static Files ใน Development

```python
# urls.py (เพิ่มใน development)
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    # ...
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
# static files ใช้ Django ในการ serve โดยอัตโนมัติเมื่อ DEBUG=True
```

---

## 6. Bootstrap 5 Integration

### base.html กับ Bootstrap 5

```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}บล็อก{% endblock %}</title>
    
    {% load static %}
    
    <!-- Bootstrap 5 CSS -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" 
          rel="stylesheet">
    <!-- Bootstrap Icons -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.0/font/bootstrap-icons.css" 
          rel="stylesheet">
    <!-- Google Fonts -->
    <link href="https://fonts.googleapis.com/css2?family=Sarabun:wght@300;400;600;700&display=swap" 
          rel="stylesheet">
    <!-- Custom CSS -->
    <link rel="stylesheet" href="{% static 'css/main.css' %}">
    
    {% block extra_css %}{% endblock %}
</head>
<body>
    <!-- Navbar -->
    <nav class="navbar navbar-expand-lg navbar-dark bg-dark">
        <div class="container">
            <a class="navbar-brand fw-bold" href="{% url 'blog:post_list' %}">
                <i class="bi bi-journal-text"></i> MyBlog
            </a>
            
            <button class="navbar-toggler" type="button" 
                    data-bs-toggle="collapse" data-bs-target="#navbarNav">
                <span class="navbar-toggler-icon"></span>
            </button>
            
            <div class="collapse navbar-collapse" id="navbarNav">
                <ul class="navbar-nav me-auto">
                    <li class="nav-item">
                        <a class="nav-link" href="{% url 'blog:post_list' %}">บทความ</a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="{% url 'pages:about' %}">เกี่ยวกับ</a>
                    </li>
                </ul>
                
                <ul class="navbar-nav">
                    {% if user.is_authenticated %}
                    <li class="nav-item">
                        <a class="nav-link" href="{% url 'blog:post_create' %}">
                            <i class="bi bi-plus-circle"></i> เขียนบทความ
                        </a>
                    </li>
                    <li class="nav-item dropdown">
                        <a class="nav-link dropdown-toggle" href="#" 
                           data-bs-toggle="dropdown">
                            <i class="bi bi-person-circle"></i> 
                            {{ user.username }}
                        </a>
                        <ul class="dropdown-menu">
                            <li>
                                <a class="dropdown-item" href="#">โปรไฟล์</a>
                            </li>
                            <li><hr class="dropdown-divider"></li>
                            <li>
                                <form method="post" action="{% url 'logout' %}" class="d-inline">
                                    {% csrf_token %}
                                    <button class="dropdown-item" type="submit">
                                        ออกจากระบบ
                                    </button>
                                </form>
                            </li>
                        </ul>
                    </li>
                    {% else %}
                    <li class="nav-item">
                        <a class="nav-link" href="{% url 'login' %}">เข้าสู่ระบบ</a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="{% url 'register' %}">สมัครสมาชิก</a>
                    </li>
                    {% endif %}
                </ul>
            </div>
        </div>
    </nav>
    
    <!-- Flash Messages -->
    {% if messages %}
    <div class="container mt-3">
        {% for message in messages %}
        <div class="alert alert-{% if message.tags == 'error' %}danger{% else %}{{ message.tags }}{% endif %} 
                    alert-dismissible fade show" role="alert">
            {{ message }}
            <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
        </div>
        {% endfor %}
    </div>
    {% endif %}
    
    <!-- Main Content -->
    <main class="container my-4">
        {% block content %}{% endblock %}
    </main>
    
    <!-- Footer -->
    <footer class="bg-dark text-light py-4 mt-5">
        <div class="container">
            <div class="row">
                <div class="col-md-6">
                    <h5>MyBlog</h5>
                    <p class="text-muted">เว็บบล็อกสำหรับแบ่งปันความรู้</p>
                </div>
                <div class="col-md-6 text-end">
                    <p class="text-muted">&copy; 2026 MyBlog. All rights reserved.</p>
                </div>
            </div>
        </div>
    </footer>
    
    <!-- Bootstrap 5 JS -->
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
    <!-- Custom JS -->
    <script src="{% static 'js/main.js' %}"></script>
    
    {% block extra_js %}{% endblock %}
</body>
</html>
```

### Blog Post List กับ Bootstrap

```html
<!-- blog/templates/blog/post_list.html -->
{% extends 'base.html' %}
{% load static %}

{% block title %}บทความทั้งหมด | MyBlog{% endblock %}

{% block content %}
<div class="row">
    <!-- Main Content -->
    <div class="col-lg-8">
        <!-- Page Header -->
        <div class="d-flex justify-content-between align-items-center mb-4">
            <h1 class="h2">บทความทั้งหมด</h1>
            {% if user.is_authenticated %}
            <a href="{% url 'blog:post_create' %}" class="btn btn-primary">
                <i class="bi bi-plus-lg"></i> เขียนบทความ
            </a>
            {% endif %}
        </div>
        
        <!-- Search Form -->
        <form method="get" class="mb-4">
            <div class="input-group">
                <input type="text" name="q" class="form-control" 
                       placeholder="ค้นหาบทความ..." 
                       value="{{ search_query }}">
                <button class="btn btn-outline-secondary" type="submit">
                    <i class="bi bi-search"></i>
                </button>
                {% if search_query %}
                <a href="{% url 'blog:post_list' %}" class="btn btn-outline-danger">
                    <i class="bi bi-x"></i>
                </a>
                {% endif %}
            </div>
        </form>
        
        {% if search_query %}
        <p class="text-muted mb-3">
            ผลการค้นหา "{{ search_query }}" - พบ {{ total_count }} บทความ
        </p>
        {% endif %}
        
        <!-- Post Cards -->
        {% for post in posts %}
        <div class="card mb-4 shadow-sm">
            <div class="row g-0">
                {% if post.featured_image %}
                <div class="col-md-4">
                    <img src="{{ post.featured_image.url }}" 
                         class="img-fluid rounded-start h-100 object-fit-cover" 
                         alt="{{ post.title }}">
                </div>
                <div class="col-md-8">
                {% else %}
                <div class="col-12">
                {% endif %}
                    <div class="card-body">
                        <!-- Category Badge -->
                        {% if post.category %}
                        <span class="badge mb-2" 
                              style="background-color: {{ post.category.color }}">
                            {{ post.category.name }}
                        </span>
                        {% endif %}
                        
                        <h5 class="card-title">
                            <a href="{{ post.get_absolute_url }}" 
                               class="text-decoration-none text-dark">
                                {{ post.title }}
                            </a>
                        </h5>
                        
                        <!-- Meta info -->
                        <p class="text-muted small">
                            <i class="bi bi-person"></i> 
                            {{ post.author.get_full_name|default:post.author.username }}
                            &nbsp;·&nbsp;
                            <i class="bi bi-calendar3"></i> 
                            {{ post.created_at|date:"d M Y" }}
                            &nbsp;·&nbsp;
                            <i class="bi bi-clock"></i> 
                            {{ post.reading_time }} นาที
                            &nbsp;·&nbsp;
                            <i class="bi bi-eye"></i> 
                            {{ post.views_count|intcomma }} ครั้ง
                        </p>
                        
                        <!-- Excerpt -->
                        {% if post.excerpt %}
                        <p class="card-text">{{ post.excerpt|truncatewords:25 }}</p>
                        {% else %}
                        <p class="card-text">{{ post.content|truncatewords:25|striptags }}</p>
                        {% endif %}
                        
                        <!-- Tags -->
                        {% if post.tags.all %}
                        <div class="mb-2">
                            {% for tag in post.tags.all %}
                            <a href="?tag={{ tag.slug }}" 
                               class="badge bg-secondary text-decoration-none me-1">
                                #{{ tag.name }}
                            </a>
                            {% endfor %}
                        </div>
                        {% endif %}
                        
                        <a href="{{ post.get_absolute_url }}" 
                           class="btn btn-sm btn-outline-primary">
                            อ่านต่อ <i class="bi bi-arrow-right"></i>
                        </a>
                    </div>
                </div>
            </div>
        </div>
        {% empty %}
        <div class="text-center py-5">
            <i class="bi bi-journal-x display-1 text-muted"></i>
            <h3 class="mt-3 text-muted">ยังไม่มีบทความ</h3>
            {% if user.is_authenticated %}
            <a href="{% url 'blog:post_create' %}" class="btn btn-primary mt-2">
                เขียนบทความแรก
            </a>
            {% endif %}
        </div>
        {% endfor %}
        
        <!-- Pagination -->
        {% if page_obj.has_other_pages %}
        <nav aria-label="Page navigation">
            <ul class="pagination justify-content-center">
                {% if page_obj.has_previous %}
                <li class="page-item">
                    <a class="page-link" 
                       href="?page={{ page_obj.previous_page_number }}{% if search_query %}&q={{ search_query }}{% endif %}">
                        <i class="bi bi-chevron-left"></i>
                    </a>
                </li>
                {% endif %}
                
                {% for num in page_obj.paginator.page_range %}
                <li class="page-item {% if page_obj.number == num %}active{% endif %}">
                    <a class="page-link" 
                       href="?page={{ num }}{% if search_query %}&q={{ search_query }}{% endif %}">
                        {{ num }}
                    </a>
                </li>
                {% endfor %}
                
                {% if page_obj.has_next %}
                <li class="page-item">
                    <a class="page-link" 
                       href="?page={{ page_obj.next_page_number }}{% if search_query %}&q={{ search_query }}{% endif %}">
                        <i class="bi bi-chevron-right"></i>
                    </a>
                </li>
                {% endif %}
            </ul>
        </nav>
        {% endif %}
    </div>
    
    <!-- Sidebar -->
    <div class="col-lg-4">
        <!-- Categories -->
        <div class="card mb-4">
            <div class="card-header">
                <h5 class="mb-0"><i class="bi bi-folder2"></i> หมวดหมู่</h5>
            </div>
            <div class="list-group list-group-flush">
                {% for category in categories %}
                <a href="?category={{ category.slug }}" 
                   class="list-group-item list-group-item-action d-flex justify-content-between {% if selected_category == category.slug %}active{% endif %}">
                    <span>
                        <span class="badge me-2" 
                              style="background-color: {{ category.color }}">&nbsp;</span>
                        {{ category.name }}
                    </span>
                    <span class="badge bg-secondary rounded-pill">
                        {{ category.posts.count }}
                    </span>
                </a>
                {% empty %}
                <div class="list-group-item text-muted">ยังไม่มีหมวดหมู่</div>
                {% endfor %}
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

---

## 7. Custom Template Tags และ Filters

```python
# blog/templatetags/__init__.py (สร้างไฟล์เปล่า)

# blog/templatetags/blog_tags.py
from django import template
from django.utils.safestring import mark_safe
from ..models import Post, Category
import markdown

register = template.Library()


# === Simple Tags ===

@register.simple_tag
def get_recent_posts(count=5):
    """
    ดึงบทความล่าสุด
    ใช้งาน: {% get_recent_posts as recent_posts %}
    หรือ: {% get_recent_posts 3 as recent_posts %}
    """
    return Post.objects.filter(
        status='published'
    ).order_by('-created_at')[:count]


@register.simple_tag
def get_categories():
    """ดึงหมวดหมู่ทั้งหมด"""
    return Category.objects.all()


@register.simple_tag(takes_context=True)
def current_url(context, view_name):
    """
    ตรวจสอบว่า URL ปัจจุบันตรงกับ view_name หรือไม่
    ใช้สำหรับ active navigation
    """
    request = context['request']
    return request.resolver_match.url_name == view_name


# === Inclusion Tags (render template) ===

@register.inclusion_tag('blog/tags/category_list.html')
def show_categories():
    """
    แสดง widget รายการหมวดหมู่
    ใช้งาน: {% show_categories %}
    """
    categories = Category.objects.all()
    return {'categories': categories}


@register.inclusion_tag('blog/tags/recent_posts.html')
def show_recent_posts(count=5):
    """แสดง widget บทความล่าสุด"""
    posts = Post.objects.filter(
        status='published'
    ).order_by('-created_at')[:count]
    return {'posts': posts}


# === Filters ===

@register.filter
def reading_time(content):
    """
    คำนวณเวลาอ่าน (นาที)
    ใช้งาน: {{ post.content|reading_time }}
    """
    word_count = len(str(content).split())
    minutes = max(1, round(word_count / 200))
    return f'{minutes} นาที'


@register.filter
def markdown_to_html(content):
    """
    แปลง Markdown เป็น HTML
    ใช้งาน: {{ post.content|markdown_to_html }}
    """
    # ติดตั้ง markdown: pip install markdown
    import markdown as md
    html = md.convert(str(content))
    return mark_safe(html)  # mark_safe บอกว่า HTML นี้ปลอดภัย


@register.filter
def shorten(value, max_length=100):
    """
    ตัดข้อความให้สั้นลง
    ใช้งาน: {{ post.content|shorten:150 }}
    """
    value = str(value)
    if len(value) > max_length:
        return value[:max_length].rsplit(' ', 1)[0] + '...'
    return value


@register.filter
def thai_date(value, format='full'):
    """
    แปลงวันที่เป็นรูปแบบไทย
    ใช้งาน: {{ post.created_at|thai_date }}
    """
    if not value:
        return ''
    
    THAI_MONTHS = {
        1: 'มกราคม', 2: 'กุมภาพันธ์', 3: 'มีนาคม',
        4: 'เมษายน', 5: 'พฤษภาคม', 6: 'มิถุนายน',
        7: 'กรกฎาคม', 8: 'สิงหาคม', 9: 'กันยายน',
        10: 'ตุลาคม', 11: 'พฤศจิกายน', 12: 'ธันวาคม'
    }
    
    thai_year = value.year + 543  # แปลงเป็นปี พ.ศ.
    month = THAI_MONTHS[value.month]
    
    if format == 'short':
        return f'{value.day}/{value.month}/{thai_year}'
    else:
        return f'{value.day} {month} {thai_year}'
```

### ใช้ Custom Template Tags

```html
<!-- โหลด tag library ก่อน -->
{% load blog_tags %}

<!-- ใช้ simple_tag -->
{% get_recent_posts as recent_posts %}
{% for post in recent_posts %}
    <a href="{{ post.get_absolute_url }}">{{ post.title }}</a>
{% endfor %}

<!-- ใช้ simple_tag กับ parameter -->
{% get_recent_posts 3 as recent_posts %}

<!-- ใช้ inclusion_tag -->
{% show_categories %}
{% show_recent_posts 5 %}

<!-- ใช้ filter -->
{{ post.content|reading_time }}
{{ post.content|markdown_to_html }}
{{ post.title|shorten:50 }}
{{ post.created_at|thai_date }}
{{ post.created_at|thai_date:"short" }}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Template Inheritance
สร้าง template hierarchy สำหรับ e-commerce:
- `base.html` - nav, footer, messages
- `shop/base.html` - extends base, เพิ่ม sidebar หมวดหมู่
- `shop/product_list.html` - extends shop/base
- `shop/product_detail.html` - extends shop/base

### แบบฝึกหัดที่ 2: Custom Template Tags
สร้าง custom template tags:
- `{% get_cart_count request %}` - นับสินค้าในตะกร้าของ user ปัจจุบัน
- `{{ price|currency:"THB" }}` - แสดงราคาพร้อม currency symbol
- `{% show_breadcrumbs %}` - แสดง breadcrumb navigation

### แบบฝึกหัดที่ 3: Bootstrap Forms
สร้าง form template ที่ใช้ Bootstrap สวยงาม:
- แสดง error messages แบบ Bootstrap alerts
- ใช้ `form-control` class
- Input validation ด้วย Bootstrap
- Submit button พร้อม loading spinner

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- Django Template Language syntax
- Template tags ทุกประเภท
- Template filters และการใช้งาน
- Template inheritance ด้วย extends/block
- Static files configuration และการใช้งาน
- Bootstrap 5 integration แบบสมบูรณ์
- Custom template tags และ filters

---

## บทถัดไป

➡️ **[Part 56: Django URLs](part-056.md)** - เรียนรู้ URL patterns, namespace, reverse(), และ include()
