# Part 062: Django Templates Advanced

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ template inheritance และการสร้าง base templates
- ใช้ template tags และ template filters ได้อย่างคล่องแคล่ว
- สร้าง custom template tags และ filters เอง
- จัดการ static files (CSS, JS, images)
- จัดการ media files (user-uploaded content)

---

## 1. Template Inheritance

Template inheritance ช่วยให้ไม่ต้องเขียน HTML ซ้ำในทุกหน้า โดยสร้าง base template แล้วให้ template อื่น extends

```html
<!-- templates/base.html - Base template หลัก -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}เว็บไซต์ของฉัน{% endblock %}</title>
    
    <!-- Static files -->
    {% load static %}
    <link rel="stylesheet" href="{% static 'css/bootstrap.min.css' %}">
    <link rel="stylesheet" href="{% static 'css/style.css' %}">
    
    <!-- Extra CSS block -->
    {% block extra_css %}{% endblock %}
</head>
<body>
    <!-- Navigation -->
    {% include 'includes/navbar.html' %}
    
    <!-- Flash messages -->
    {% if messages %}
    <div class="container mt-3">
        {% for message in messages %}
        <div class="alert alert-{{ message.tags }} alert-dismissible fade show">
            {{ message }}
            <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
        </div>
        {% endfor %}
    </div>
    {% endif %}
    
    <!-- Main content block -->
    <main class="container my-4">
        {% block content %}
        <!-- เนื้อหาหลักจะอยู่ตรงนี้ -->
        {% endblock %}
    </main>
    
    <!-- Footer -->
    {% include 'includes/footer.html' %}
    
    <!-- Scripts -->
    <script src="{% static 'js/bootstrap.bundle.min.js' %}"></script>
    {% block extra_js %}{% endblock %}
</body>
</html>
```

```html
<!-- templates/includes/navbar.html -->
<nav class="navbar navbar-expand-lg navbar-dark bg-dark">
    <div class="container">
        <a class="navbar-brand" href="{% url 'home' %}">MyApp</a>
        
        <div class="navbar-nav ms-auto">
            {% if user.is_authenticated %}
            <a class="nav-link" href="{% url 'dashboard' %}">Dashboard</a>
            <a class="nav-link" href="{% url 'logout' %}">ออกจากระบบ</a>
            {% else %}
            <a class="nav-link" href="{% url 'login' %}">เข้าสู่ระบบ</a>
            <a class="nav-link" href="{% url 'register' %}">สมัครสมาชิก</a>
            {% endif %}
        </div>
    </div>
</nav>
```

```html
<!-- templates/home.html - Child template -->
{% extends 'base.html' %}
{% load static %}

<!-- กำหนด title -->
{% block title %}หน้าหลัก - MyApp{% endblock %}

<!-- เพิ่ม CSS เฉพาะหน้านี้ -->
{% block extra_css %}
<link rel="stylesheet" href="{% static 'css/home.css' %}">
{% endblock %}

<!-- เนื้อหาหลัก -->
{% block content %}
<div class="row">
    <div class="col-md-8">
        <h1>ยินดีต้อนรับ!</h1>
        <p>นี่คือหน้าหลักของเว็บไซต์</p>
        
        {% for article in articles %}
        <div class="card mb-3">
            <div class="card-body">
                <h5 class="card-title">{{ article.title }}</h5>
                <p class="card-text">{{ article.excerpt }}</p>
                <a href="{{ article.get_absolute_url }}" class="btn btn-primary">อ่านต่อ</a>
            </div>
        </div>
        {% empty %}
        <p>ยังไม่มีบทความ</p>
        {% endfor %}
    </div>
    
    <div class="col-md-4">
        {% block sidebar %}
        <!-- sidebar สำหรับ override ใน subclass -->
        {% endblock %}
    </div>
</div>
{% endblock %}

<!-- เพิ่ม JS เฉพาะหน้านี้ -->
{% block extra_js %}
<script src="{% static 'js/home.js' %}"></script>
{% endblock %}
```

```html
<!-- templates/articles/detail.html - นำไปใช้งาน -->
{% extends 'base.html' %}

{% block title %}{{ article.title }} - MyApp{% endblock %}

{% block content %}
<article>
    <h1>{{ article.title }}</h1>
    <div class="meta">
        เขียนโดย {{ article.author.get_full_name }} 
        เมื่อ {{ article.created_at|date:"d M Y" }}
    </div>
    
    <!-- แสดง content ที่มี HTML -->
    <div class="content">
        {{ article.content|safe }}
    </div>
</article>
{% endblock %}

{% block sidebar %}
<!-- Override sidebar สำหรับหน้านี้ -->
<h5>บทความที่เกี่ยวข้อง</h5>
{% for related in related_articles %}
<a href="{{ related.get_absolute_url }}">{{ related.title }}</a>
{% endfor %}
{% endblock %}
```

---

## 2. Template Tags

### Built-in Template Tags

```html
<!-- if/elif/else -->
{% if user.is_authenticated %}
    <p>สวัสดี {{ user.username }}</p>
{% elif user.is_anonymous %}
    <p>กรุณาเข้าสู่ระบบ</p>
{% else %}
    <p>ไม่ทราบสถานะ</p>
{% endif %}

<!-- เงื่อนไขซับซ้อน -->
{% if score >= 80 and user.is_active %}
<p>ผ่านการทดสอบ</p>
{% endif %}

{% if not user.is_staff %}
<p>คุณไม่มีสิทธิ์เข้าถึงส่วนนี้</p>
{% endif %}

<!-- for loop -->
{% for item in items %}
    <p>{{ forloop.counter }}: {{ item.name }}</p>  {# นับจาก 1 #}
    <p>{{ forloop.counter0 }}: {{ item.name }}</p> {# นับจาก 0 #}
    <p>{{ forloop.revcounter }}</p>  {# นับถอยหลัง #}
    <p>{{ forloop.first }}</p>       {# True ถ้าเป็น item แรก #}
    <p>{{ forloop.last }}</p>        {# True ถ้าเป็น item สุดท้าย #}
    <p>{{ forloop.parentloop.counter }}</p>  {# loop นอก #}
{% empty %}
    <p>ไม่มีข้อมูล</p>
{% endfor %}

<!-- for loop กับ dict -->
{% for key, value in my_dict.items %}
<p>{{ key }}: {{ value }}</p>
{% endfor %}

<!-- with - กำหนด alias ให้ตัวแปร -->
{% with article.author.get_full_name as author_name %}
<p>เขียนโดย: {{ author_name }}</p>
{% endwith %}

<!-- url - สร้าง URL จาก URL name -->
<a href="{% url 'article_detail' pk=article.pk %}">{{ article.title }}</a>
<a href="{% url 'profile' username=user.username %}">โปรไฟล์</a>

<!-- block/extends ดูส่วน Template Inheritance -->

<!-- include - รวม template อื่น -->
{% include 'partials/pagination.html' %}
{% include 'partials/card.html' with article=article show_author=True %}

<!-- comment - ความคิดเห็น -->
{# ความคิดเห็นบรรทัดเดียว - ไม่แสดงใน output #}

{% comment "optional reason" %}
ความคิดเห็นหลายบรรทัด
ไม่แสดงใน rendered HTML
{% endcomment %}

<!-- verbatim - แสดง template syntax โดยไม่ process -->
{% verbatim %}
{{ this will not be processed }}
{% for item in items %}{{ item }}{% endfor %}
{% endverbatim %}

<!-- spaceless - ลบ whitespace ระหว่าง HTML tags -->
{% spaceless %}
<p>  hello  </p>
  <span>world</span>
{% endspaceless %}
```

---

## 3. Template Filters

```html
<!-- String filters -->
{{ name|upper }}                    {# JOHN DOE #}
{{ name|lower }}                    {# john doe #}
{{ name|title }}                    {# John Doe #}
{{ name|capfirst }}                 {# John doe (ตัวแรกพิมพ์ใหญ่) #}
{{ name|length }}                   {# 8 #}
{{ text|truncatechars:50 }}         {# ตัดที่ 50 ตัวอักษร... #}
{{ text|truncatewords:10 }}         {# ตัดที่ 10 คำ... #}
{{ text|truncatechars_html:100 }}   {# ตัดโดยรักษา HTML tags #}
{{ text|wordcount }}                {# นับจำนวนคำ #}
{{ text|linebreaks }}               {# แปลง \n เป็น <p> และ <br> #}
{{ text|linebreaksbr }}             {# แปลง \n เป็น <br> #}
{{ text|striptags }}                {# ลบ HTML tags ทั้งหมด #}
{{ text|safe }}                     {# mark as safe HTML (ระวัง XSS!) #}
{{ text|escape }}                   {# HTML escape: < becomes &lt; #}
{{ text|slugify }}                  {# แปลงเป็น slug #}
{{ text|urlencode }}                {# URL encode #}
{{ url|urlize }}                    {# แปลง URL เป็น <a> link #}

<!-- Number filters -->
{{ price|floatformat:2 }}           {# 1234.57 (2 ทศนิยม) #}
{{ price|floatformat:-2 }}          {# ลบ trailing zeros #}
{{ number|intcomma }}               {# 1,234,567 #}
{{ number|filesizeformat }}         {# 1.2 MB #}
{{ number|ordinal }}                {# 1st, 2nd, 3rd #}

<!-- Date filters -->
{{ date|date:"d/m/Y" }}             {# 15/01/2024 #}
{{ date|date:"D, d M Y" }}          {# Tue, 15 Jan 2024 #}
{{ date|date:"d M Y, H:i" }}        {# 15 Jan 2024, 14:30 #}
{{ date|time:"H:i" }}               {# 14:30 #}
{{ date|timesince }}                {# 3 ชั่วโมงที่แล้ว #}
{{ date|timeuntil }}                {# 2 วันข้างหน้า #}

<!-- List/Dict filters -->
{{ items|length }}                  {# จำนวน items #}
{{ items|first }}                   {# item แรก #}
{{ items|last }}                    {# item สุดท้าย #}
{{ items|join:", " }}               {# a, b, c #}
{{ items|slice:":5" }}              {# 5 items แรก #}
{{ items|dictsort:"name" }}         {# เรียง dict ตาม key #}
{{ items|random }}                  {# random item #}

<!-- Default filters -->
{{ value|default:"ค่าเริ่มต้น" }}    {# ถ้า value เป็น falsy #}
{{ value|default_if_none:"N/A" }}   {# ถ้า value เป็น None #}

<!-- Yesno filter -->
{{ is_active|yesno:"ใช้งาน,ไม่ใช้งาน,ไม่ทราบ" }}

<!-- Add filter -->
{{ number|add:10 }}                 {# บวก 10 #}
{{ list1|add:list2 }}               {# รวม list #}
```

---

## 4. Custom Template Tags

```
# โครงสร้างไฟล์
myapp/
├── templatetags/          # ต้องสร้าง directory นี้
│   ├── __init__.py        # ต้องมีไฟล์นี้
│   └── myapp_tags.py      # custom tags
├── models.py
├── views.py
└── ...
```

```python
# myapp/templatetags/myapp_tags.py
from django import template
from django.utils.html import format_html
from django.utils.safestring import mark_safe

register = template.Library()  # สร้าง Library instance

# =======================
# Simple Tags
# =======================

@register.simple_tag
def current_time(format_string='%H:%M:%S'):
    """Tag แสดงเวลาปัจจุบัน
    
    ใช้งาน: {% current_time %}
    หรือ: {% current_time "%d/%m/%Y %H:%M" %}
    """
    from datetime import datetime
    return datetime.now().strftime(format_string)


@register.simple_tag(takes_context=True)
def active_link(context, url_name, css_class='active'):
    """Tag ตรวจสอบว่า link ปัจจุบัน active หรือไม่
    
    ใช้งาน: {% active_link 'home' %}
    """
    request = context['request']
    from django.urls import reverse
    try:
        url = reverse(url_name)
        if request.path == url:
            return css_class
    except Exception:
        pass
    return ''


@register.simple_tag
def get_setting(setting_name):
    """Tag ดึงค่า settings
    
    ใช้งาน: {% get_setting 'SITE_NAME' %}
    """
    from django.conf import settings
    return getattr(settings, setting_name, '')


# =======================
# Inclusion Tags
# =======================

@register.inclusion_tag('includes/pagination.html')
def show_pagination(page_obj, url_name='article_list'):
    """Tag แสดง pagination
    
    ใช้งาน: {% show_pagination page_obj %}
    """
    return {
        'page_obj': page_obj,
        'url_name': url_name,
    }


@register.inclusion_tag('includes/breadcrumb.html', takes_context=True)
def breadcrumb(context, *args):
    """Tag แสดง breadcrumb
    
    ใช้งาน: {% breadcrumb 'หน้าหลัก' '/' 'บทความ' '/articles/' 'หัวข้อบทความ' %}
    """
    items = []
    for i in range(0, len(args) - 1, 2):
        items.append({
            'label': args[i],
            'url': args[i + 1] if i + 1 < len(args) else None
        })
    if len(args) % 2 != 0:
        items.append({'label': args[-1], 'url': None})
    
    return {'breadcrumb_items': items}


# =======================
# Assignment Tags
# =======================

@register.simple_tag
def get_recent_articles(count=5):
    """Tag ดึงบทความล่าสุด และเก็บในตัวแปร
    
    ใช้งาน: {% get_recent_articles 5 as recent_articles %}
    """
    from myapp.models import Article
    return Article.objects.filter(is_published=True).order_by('-created_at')[:count]


# วิธีใช้ as:
# {% get_recent_articles 3 as latest %}
# {% for article in latest %}...{% endfor %}


# =======================
# Filters
# =======================

@register.filter
def thai_date(value):
    """Filter แปลง date เป็น พ.ศ.
    
    ใช้งาน: {{ article.created_at|thai_date }}
    """
    if not value:
        return ''
    
    thai_months = [
        'มกราคม', 'กุมภาพันธ์', 'มีนาคม', 'เมษายน',
        'พฤษภาคม', 'มิถุนายน', 'กรกฎาคม', 'สิงหาคม',
        'กันยายน', 'ตุลาคม', 'พฤศจิกายน', 'ธันวาคม'
    ]
    
    day = value.day
    month = thai_months[value.month - 1]
    year = value.year + 543  # แปลงเป็น พ.ศ.
    
    return f'{day} {month} {year}'


@register.filter
def thai_number(value):
    """Filter แปลงตัวเลขเป็นตัวเลขไทย
    
    ใช้งาน: {{ count|thai_number }}
    """
    thai_digits = '๐๑๒๓๔๕๖๗๘๙'
    result = ''
    for char in str(value):
        if char.isdigit():
            result += thai_digits[int(char)]
        else:
            result += char
    return result


@register.filter
def multiply(value, arg):
    """Filter คูณตัวเลข
    
    ใช้งาน: {{ price|multiply:1.07 }}
    """
    try:
        return float(value) * float(arg)
    except (TypeError, ValueError):
        return ''


@register.filter
def truncate_html(value, length=100):
    """Filter ตัดข้อความ HTML อย่างปลอดภัย"""
    from django.utils.html import strip_tags
    text = strip_tags(value)
    if len(text) <= length:
        return value
    return text[:length] + '...'


# =======================
# Block Tags (ซับซ้อนกว่า)
# =======================

class HighlightNode(template.Node):
    """Node สำหรับ highlight text"""
    
    def __init__(self, text, search_term):
        self.text = template.Variable(text)
        self.search_term = template.Variable(search_term)
    
    def render(self, context):
        text = self.text.resolve(context)
        search_term = self.search_term.resolve(context)
        
        highlighted = str(text).replace(
            search_term,
            f'<mark>{search_term}</mark>'
        )
        return mark_safe(highlighted)


@register.tag
def highlight(parser, token):
    """Tag highlight text
    
    ใช้งาน: {% highlight article.content search_query %}
    """
    try:
        tag_name, text, search_term = token.split_contents()
    except ValueError:
        raise template.TemplateSyntaxError(
            f'"{token.contents.split()[0]}" tag requires 2 arguments'
        )
    return HighlightNode(text, search_term)
```

### การใช้ Custom Tags ใน Template

```html
<!-- templates/articles/list.html -->
{% extends 'base.html' %}
{% load myapp_tags %}  {# โหลด custom tags #}

{% block title %}บทความ - {% get_setting 'SITE_NAME' %}{% endblock %}

{% block content %}
<h1>บทความทั้งหมด</h1>

<!-- แสดงเวลาปัจจุบัน -->
<p>อัปเดตล่าสุด: {% current_time "%d/%m/%Y %H:%M" %}</p>

<!-- บทความล่าสุดใน sidebar -->
{% get_recent_articles 5 as recent %}

{% for article in articles %}
<div class="card">
    <h5>{{ article.title }}</h5>
    <!-- ใช้ thai_date filter -->
    <small>{{ article.created_at|thai_date }}</small>
    <!-- ใช้ truncate_html filter -->
    <p>{{ article.content|truncate_html:200 }}</p>
</div>
{% endfor %}

<!-- Pagination -->
{% show_pagination page_obj %}

{% endblock %}
```

---

## 5. Static Files

```python
# settings.py
import os

# Static files (CSS, JavaScript, Images)
STATIC_URL = '/static/'

# Directories ที่ Django หา static files ใน development
STATICFILES_DIRS = [
    os.path.join(BASE_DIR, 'static'),    # global static directory
]

# Directory ที่เก็บ static files หลัง collectstatic
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')

# ตัวเลือก static file finders
STATICFILES_FINDERS = [
    'django.contrib.staticfiles.finders.FileSystemFinder',
    'django.contrib.staticfiles.finders.AppDirectoriesFinder',
]
```

```
# โครงสร้าง static files
project/
├── static/                    # Global static files
│   ├── css/
│   │   ├── style.css
│   │   └── bootstrap.min.css
│   ├── js/
│   │   ├── main.js
│   │   └── bootstrap.bundle.min.js
│   └── images/
│       └── logo.png
├── myapp/
│   └── static/               # App-specific static files
│       └── myapp/
│           ├── css/
│           └── js/
└── staticfiles/               # collectstatic output
```

```html
<!-- ใช้ {% load static %} ก่อนเสมอ -->
{% load static %}

<!-- CSS -->
<link rel="stylesheet" href="{% static 'css/style.css' %}">

<!-- JavaScript -->
<script src="{% static 'js/main.js' %}"></script>

<!-- Image -->
<img src="{% static 'images/logo.png' %}" alt="Logo">

<!-- เก็บ URL ในตัวแปร -->
{% static 'images/logo.png' as logo_url %}
<img src="{{ logo_url }}" alt="Logo">
```

### Whitenoise สำหรับ Production

```bash
pip install whitenoise
```

```python
# settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',  # เพิ่มต่อจาก SecurityMiddleware
    # ...
]

# กำหนด compression และ caching
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
```

---

## 6. Media Files

```python
# settings.py
MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')
```

```python
# urls.py (development only)
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    # ...
]

if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

```html
<!-- template - แสดงรูปภาพที่ user upload -->
{% if article.thumbnail %}
<img src="{{ article.thumbnail.url }}" alt="{{ article.title }}">
{% else %}
<img src="{% static 'images/default-thumbnail.jpg' %}" alt="Default">
{% endif %}
```

---

## 7. Template Context Processors

```python
# settings.py
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [os.path.join(BASE_DIR, 'templates')],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
                # Custom context processor
                'myapp.context_processors.site_settings',
            ],
        },
    },
]
```

```python
# myapp/context_processors.py
def site_settings(request):
    """ส่ง settings ไปให้ทุก template โดยอัตโนมัติ"""
    from django.conf import settings
    return {
        'SITE_NAME': getattr(settings, 'SITE_NAME', 'My Site'),
        'SITE_DESCRIPTION': getattr(settings, 'SITE_DESCRIPTION', ''),
        'CONTACT_EMAIL': getattr(settings, 'CONTACT_EMAIL', ''),
    }


def cart_count(request):
    """ส่งจำนวนสินค้าใน cart ไปทุก template"""
    count = 0
    if request.user.is_authenticated:
        from shop.models import Cart
        try:
            cart = Cart.objects.get(user=request.user)
            count = cart.items.count()
        except Cart.DoesNotExist:
            pass
    return {'cart_count': count}
```

---

## 8. สรุป Part 062

✅ **Template Inheritance** ใช้ `extends` และ `block` ลดการเขียนโค้ดซ้ำ
✅ **Template Tags** มีทั้ง built-in (`if`, `for`, `url`, `include`) และ custom
✅ **Template Filters** แปลงข้อมูลก่อนแสดงผล เช่น `|date`, `|truncatechars`
✅ **Custom Template Tags** สร้างด้วย `@register.simple_tag`, `@register.inclusion_tag`, `@register.filter`
✅ **Static Files** ใช้ `{% load static %}` และ `{% static 'path/to/file' %}`
✅ **Media Files** ใช้ `MEDIA_URL` และ `MEDIA_ROOT` สำหรับไฟล์ที่ user upload
✅ **Context Processors** ส่งข้อมูลไปทุก template โดยอัตโนมัติ

## ➡️ ถัดไป: Part 063 - Django Authentication

*Part 062/100+ | Python Course - Beginner to World-Class*
