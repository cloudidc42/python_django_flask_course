# Part 061: Django Forms

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจการสร้างและใช้งาน Django Forms
- สร้าง ModelForm เพื่อเชื่อมต่อกับ Model
- ทำ form validation และ custom validators
- Render form ใน templates อย่างถูกต้อง
- เข้าใจ CSRF protection
- รองรับการ upload ไฟล์ผ่าน form

---

## 1. Django Forms คืออะไร?

Django มี form system ที่ทรงพลังซึ่งช่วยจัดการ:
- HTML form generation
- Data validation
- Data cleaning และ normalization
- Error message handling

Forms ใน Django แบ่งได้เป็น 2 ประเภทหลัก:
1. **`forms.Form`** - Form ทั่วไปที่สร้าง field เอง
2. **`forms.ModelForm`** - Form ที่สร้างจาก Model อัตโนมัติ

---

## 2. การสร้าง Form class พื้นฐาน

```python
# forms.py
from django import forms

class ContactForm(forms.Form):
    """Form สำหรับหน้าติดต่อ"""
    
    # ฟิลด์ชื่อ - required โดย default
    name = forms.CharField(
        max_length=100,
        label='ชื่อ-นามสกุล',
        widget=forms.TextInput(attrs={
            'class': 'form-control',
            'placeholder': 'กรอกชื่อของคุณ'
        })
    )
    
    # ฟิลด์อีเมล
    email = forms.EmailField(
        label='อีเมล',
        widget=forms.EmailInput(attrs={
            'class': 'form-control',
            'placeholder': 'example@email.com'
        })
    )
    
    # ฟิลด์เบอร์โทร - ไม่บังคับ
    phone = forms.CharField(
        max_length=20,
        required=False,
        label='เบอร์โทรศัพท์',
        help_text='ไม่บังคับ'
    )
    
    # ฟิลด์หัวข้อ - เลือกจากรายการ
    subject = forms.ChoiceField(
        choices=[
            ('general', 'ทั่วไป'),
            ('support', 'ขอความช่วยเหลือ'),
            ('complaint', 'ร้องเรียน'),
            ('suggestion', 'ข้อเสนอแนะ'),
        ],
        label='หัวข้อ'
    )
    
    # ฟิลด์ข้อความ
    message = forms.CharField(
        widget=forms.Textarea(attrs={
            'class': 'form-control',
            'rows': 5,
            'placeholder': 'กรอกข้อความของคุณ'
        }),
        label='ข้อความ',
        min_length=10,
        max_length=1000
    )
    
    # ฟิลด์ checkbox
    newsletter = forms.BooleanField(
        required=False,
        label='รับข่าวสารทางอีเมล',
        initial=True
    )
```

### Field Types ที่สำคัญ

```python
from django import forms

class FieldDemoForm(forms.Form):
    # Text fields
    char_field = forms.CharField()           # ข้อความสั้น
    text_area = forms.CharField(widget=forms.Textarea)  # ข้อความยาว
    email_field = forms.EmailField()         # อีเมล
    url_field = forms.URLField()             # URL
    slug_field = forms.SlugField()           # slug
    
    # Number fields
    integer_field = forms.IntegerField()     # จำนวนเต็ม
    float_field = forms.FloatField()         # จำนวนทศนิยม
    decimal_field = forms.DecimalField(      # ทศนิยมแม่นยำ
        max_digits=10,
        decimal_places=2
    )
    
    # Date/Time fields
    date_field = forms.DateField()           # วันที่
    time_field = forms.TimeField()           # เวลา
    datetime_field = forms.DateTimeField()   # วันที่และเวลา
    
    # Choice fields
    choice_field = forms.ChoiceField(choices=[('a', 'ตัวเลือก A'), ('b', 'ตัวเลือก B')])
    multiple_choice = forms.MultipleChoiceField(choices=[('a', 'A'), ('b', 'B'), ('c', 'C')])
    
    # Boolean fields
    bool_field = forms.BooleanField()        # checkbox
    null_bool_field = forms.NullBooleanField()  # Yes/No/Unknown
    
    # File fields
    file_field = forms.FileField()           # ไฟล์ทั่วไป
    image_field = forms.ImageField()         # ไฟล์รูปภาพ
```

---

## 3. การใช้ Form ใน View

```python
# views.py
from django.shortcuts import render, redirect
from django.contrib import messages
from .forms import ContactForm

def contact_view(request):
    """View สำหรับหน้าติดต่อ"""
    
    if request.method == 'POST':
        # รับข้อมูลจาก POST request
        form = ContactForm(request.POST)
        
        # ตรวจสอบความถูกต้องของข้อมูล
        if form.is_valid():
            # ดึงข้อมูลที่ผ่านการ validate แล้ว
            name = form.cleaned_data['name']
            email = form.cleaned_data['email']
            subject = form.cleaned_data['subject']
            message = form.cleaned_data['message']
            newsletter = form.cleaned_data['newsletter']
            
            # ส่งอีเมลหรือบันทึกข้อมูล
            # send_contact_email(name, email, subject, message)
            
            # แจ้งผู้ใช้
            messages.success(request, f'ขอบคุณ {name}! ส่งข้อความเรียบร้อยแล้ว')
            return redirect('contact_success')
        # ถ้าข้อมูลไม่ถูกต้อง form จะมี errors อัตโนมัติ
        
    else:
        # GET request - สร้าง form ว่าง
        form = ContactForm()
    
    return render(request, 'contact/contact.html', {'form': form})
```

---

## 4. การ Render Form ใน Template

```html
<!-- templates/contact/contact.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>ติดต่อเรา</title>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css">
</head>
<body>
<div class="container mt-4">
    <h1>ติดต่อเรา</h1>
    
    <!-- แสดง messages -->
    {% for message in messages %}
    <div class="alert alert-{{ message.tags }}">{{ message }}</div>
    {% endfor %}
    
    <form method="post" enctype="multipart/form-data">
        <!-- CSRF token - สำคัญมาก! -->
        {% csrf_token %}
        
        <!-- วิธีที่ 1: Render ทั้ง form อัตโนมัติ -->
        {{ form.as_p }}
        
        <button type="submit" class="btn btn-primary">ส่งข้อความ</button>
    </form>
</div>
</body>
</html>
```

### วิธี Render Form แบบต่างๆ

```html
<!-- วิธีที่ 2: Render เป็น table -->
<form method="post">
    {% csrf_token %}
    <table>
        {{ form.as_table }}
    </table>
    <button type="submit">ส่ง</button>
</form>

<!-- วิธีที่ 3: Render เป็น list -->
<form method="post">
    {% csrf_token %}
    <ul>
        {{ form.as_ul }}
    </ul>
    <button type="submit">ส่ง</button>
</form>

<!-- วิธีที่ 4: Render แต่ละ field เอง (แนะนำสำหรับ custom layout) -->
<form method="post">
    {% csrf_token %}
    
    <div class="mb-3">
        <label for="{{ form.name.id_for_label }}" class="form-label">
            {{ form.name.label }}
        </label>
        {{ form.name }}
        <!-- แสดง errors ของ field นี้ -->
        {% if form.name.errors %}
        <div class="invalid-feedback d-block">
            {% for error in form.name.errors %}
            {{ error }}
            {% endfor %}
        </div>
        {% endif %}
        <!-- แสดง help text -->
        {% if form.name.help_text %}
        <small class="form-text text-muted">{{ form.name.help_text }}</small>
        {% endif %}
    </div>
    
    <div class="mb-3">
        <label for="{{ form.email.id_for_label }}" class="form-label">
            {{ form.email.label }}
        </label>
        {{ form.email }}
        {% if form.email.errors %}
        <div class="invalid-feedback d-block">
            {% for error in form.email.errors %}{{ error }}{% endfor %}
        </div>
        {% endif %}
    </div>
    
    <!-- แสดง non-field errors (errors ที่ไม่เกี่ยวกับ field ใด field หนึ่ง) -->
    {% if form.non_field_errors %}
    <div class="alert alert-danger">
        {% for error in form.non_field_errors %}
        <p>{{ error }}</p>
        {% endfor %}
    </div>
    {% endif %}
    
    <button type="submit" class="btn btn-primary">ส่งข้อความ</button>
</form>
```

---

## 5. ModelForm

ModelForm สร้าง form จาก Model โดยอัตโนมัติ ลดการเขียนโค้ดซ้ำ

```python
# models.py
from django.db import models

class Article(models.Model):
    """Model บทความ"""
    title = models.CharField(max_length=200, verbose_name='หัวข้อ')
    slug = models.SlugField(unique=True, verbose_name='Slug')
    author = models.ForeignKey(
        'auth.User',
        on_delete=models.CASCADE,
        verbose_name='ผู้เขียน'
    )
    category = models.ForeignKey(
        'Category',
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        verbose_name='หมวดหมู่'
    )
    content = models.TextField(verbose_name='เนื้อหา')
    excerpt = models.TextField(max_length=500, blank=True, verbose_name='บทคัดย่อ')
    thumbnail = models.ImageField(
        upload_to='articles/thumbnails/',
        null=True,
        blank=True,
        verbose_name='รูปหน้าปก'
    )
    is_published = models.BooleanField(default=False, verbose_name='เผยแพร่')
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        verbose_name = 'บทความ'
        verbose_name_plural = 'บทความ'
    
    def __str__(self):
        return self.title
```

```python
# forms.py
from django import forms
from .models import Article

class ArticleForm(forms.ModelForm):
    """Form สำหรับสร้าง/แก้ไขบทความ"""
    
    class Meta:
        model = Article
        # fields = '__all__'  # ใช้ทุก field (ไม่แนะนำ - security risk)
        fields = ['title', 'slug', 'category', 'content', 'excerpt', 'thumbnail', 'is_published']
        # exclude = ['author', 'created_at', 'updated_at']  # ยกเว้น fields
        
        # กำหนด labels
        labels = {
            'title': 'หัวข้อบทความ',
            'slug': 'URL Slug',
            'category': 'หมวดหมู่',
            'content': 'เนื้อหา',
            'excerpt': 'บทคัดย่อ',
            'thumbnail': 'รูปหน้าปก',
            'is_published': 'เผยแพร่ทันที',
        }
        
        # กำหนด help texts
        help_texts = {
            'slug': 'ใช้สำหรับ URL เช่น my-first-article',
            'excerpt': 'ข้อความสั้นๆ สำหรับแสดงในหน้าหลัก',
        }
        
        # กำหนด widgets
        widgets = {
            'title': forms.TextInput(attrs={
                'class': 'form-control',
                'placeholder': 'กรอกหัวข้อบทความ'
            }),
            'slug': forms.TextInput(attrs={
                'class': 'form-control',
                'placeholder': 'my-article-slug'
            }),
            'content': forms.Textarea(attrs={
                'class': 'form-control',
                'rows': 10,
                'id': 'content-editor'
            }),
            'excerpt': forms.Textarea(attrs={
                'class': 'form-control',
                'rows': 3
            }),
            'is_published': forms.CheckboxInput(attrs={
                'class': 'form-check-input'
            }),
        }
        
        # กำหนด error messages
        error_messages = {
            'title': {
                'required': 'กรุณากรอกหัวข้อบทความ',
                'max_length': 'หัวข้อยาวเกินไป (ไม่เกิน 200 ตัวอักษร)',
            },
            'slug': {
                'unique': 'Slug นี้ถูกใช้แล้ว กรุณาเลือก slug อื่น',
            },
        }
    
    def __init__(self, *args, **kwargs):
        """กำหนดค่าเริ่มต้น"""
        super().__init__(*args, **kwargs)
        # เพิ่ม class ให้ทุก field อัตโนมัติ
        for field_name, field in self.fields.items():
            if not isinstance(field.widget, forms.CheckboxInput):
                if 'class' not in field.widget.attrs:
                    field.widget.attrs['class'] = 'form-control'
```

### การใช้ ModelForm ใน View

```python
# views.py
from django.shortcuts import render, redirect, get_object_or_404
from django.contrib.auth.decorators import login_required
from django.contrib import messages
from .models import Article
from .forms import ArticleForm

@login_required
def article_create(request):
    """สร้างบทความใหม่"""
    if request.method == 'POST':
        form = ArticleForm(request.POST, request.FILES)  # request.FILES สำหรับไฟล์
        if form.is_valid():
            # บันทึก object แต่ยังไม่ commit ลง database
            article = form.save(commit=False)
            # กำหนด author เป็น user ที่ login อยู่
            article.author = request.user
            # บันทึกลง database จริง
            article.save()
            # บันทึก many-to-many fields (ถ้ามี)
            form.save_m2m()
            
            messages.success(request, 'สร้างบทความเรียบร้อยแล้ว!')
            return redirect('article_detail', pk=article.pk)
    else:
        form = ArticleForm()
    
    return render(request, 'articles/article_form.html', {'form': form, 'action': 'สร้าง'})

@login_required
def article_edit(request, pk):
    """แก้ไขบทความ"""
    article = get_object_or_404(Article, pk=pk, author=request.user)
    
    if request.method == 'POST':
        # ส่ง instance เพื่อ update แทน create
        form = ArticleForm(request.POST, request.FILES, instance=article)
        if form.is_valid():
            form.save()
            messages.success(request, 'อัปเดตบทความเรียบร้อยแล้ว!')
            return redirect('article_detail', pk=article.pk)
    else:
        # populate form ด้วยข้อมูลเดิม
        form = ArticleForm(instance=article)
    
    return render(request, 'articles/article_form.html', {'form': form, 'action': 'แก้ไข'})
```

---

## 6. Form Validation

### Built-in Validation

```python
from django import forms

class RegistrationForm(forms.Form):
    username = forms.CharField(
        min_length=3,       # ความยาวขั้นต่ำ
        max_length=50,      # ความยาวสูงสุด
    )
    email = forms.EmailField()  # ตรวจสอบรูปแบบ email อัตโนมัติ
    age = forms.IntegerField(
        min_value=18,       # ค่าต่ำสุด
        max_value=120,      # ค่าสูงสุด
    )
    website = forms.URLField(required=False)
    password = forms.CharField(
        min_length=8,
        widget=forms.PasswordInput
    )
    confirm_password = forms.CharField(widget=forms.PasswordInput)
```

### Field-level Validation (clean_<fieldname>)

```python
class RegistrationForm(forms.Form):
    username = forms.CharField(min_length=3, max_length=50)
    email = forms.EmailField()
    password = forms.CharField(min_length=8, widget=forms.PasswordInput)
    confirm_password = forms.CharField(widget=forms.PasswordInput)
    
    def clean_username(self):
        """Validate field username"""
        username = self.cleaned_data.get('username')
        
        # ตรวจสอบ username ซ้ำในฐานข้อมูล
        from django.contrib.auth.models import User
        if User.objects.filter(username=username).exists():
            raise forms.ValidationError('Username นี้ถูกใช้แล้ว')
        
        # ตรวจสอบตัวอักษรพิเศษ
        import re
        if not re.match(r'^[a-zA-Z0-9_]+$', username):
            raise forms.ValidationError('Username ใช้ได้เฉพาะตัวอักษร ตัวเลข และ _')
        
        # คืนค่า cleaned data เสมอ
        return username.lower()  # แปลงเป็นตัวพิมพ์เล็ก
    
    def clean_email(self):
        """Validate field email"""
        email = self.cleaned_data.get('email')
        
        from django.contrib.auth.models import User
        if User.objects.filter(email=email).exists():
            raise forms.ValidationError('อีเมลนี้ถูกใช้แล้ว')
        
        # ตรวจสอบ domain ที่อนุญาต
        blocked_domains = ['spam.com', 'trash.com']
        domain = email.split('@')[1]
        if domain in blocked_domains:
            raise forms.ValidationError('ไม่อนุญาตให้ใช้อีเมลจาก domain นี้')
        
        return email
    
    def clean(self):
        """Validate ข้ามหลาย fields (cross-field validation)"""
        cleaned_data = super().clean()
        password = cleaned_data.get('password')
        confirm_password = cleaned_data.get('confirm_password')
        
        if password and confirm_password:
            if password != confirm_password:
                # เพิ่ม error ให้ field ใดก็ได้
                self.add_error('confirm_password', 'รหัสผ่านไม่ตรงกัน')
                # หรือ raise ValidationError เพื่อเป็น non_field_error
                # raise forms.ValidationError('รหัสผ่านไม่ตรงกัน')
        
        return cleaned_data
```

---

## 7. Custom Validators

```python
# validators.py
from django.core.exceptions import ValidationError
import re

def validate_thai_phone(value):
    """Validator สำหรับเบอร์โทรศัพท์ไทย"""
    # ลบ space และ dash
    phone = re.sub(r'[\s\-]', '', value)
    
    # เช็คว่าเป็นตัวเลขทั้งหมด
    if not phone.isdigit():
        raise ValidationError('เบอร์โทรต้องเป็นตัวเลขเท่านั้น')
    
    # เช็คความยาว
    if len(phone) not in [9, 10]:
        raise ValidationError('เบอร์โทรต้องมี 9-10 หลัก')
    
    # เช็ค prefix ของมือถือไทย
    mobile_prefixes = ['06', '08', '09']
    landline_prefixes = ['02', '03', '04', '05', '07']
    
    if len(phone) == 10 and phone[:2] not in mobile_prefixes:
        raise ValidationError('เบอร์มือถือต้องขึ้นต้นด้วย 06, 08, หรือ 09')


def validate_strong_password(value):
    """Validator สำหรับรหัสผ่านที่แข็งแกร่ง"""
    if len(value) < 8:
        raise ValidationError('รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร')
    
    if not re.search(r'[A-Z]', value):
        raise ValidationError('รหัสผ่านต้องมีตัวอักษรพิมพ์ใหญ่อย่างน้อย 1 ตัว')
    
    if not re.search(r'[a-z]', value):
        raise ValidationError('รหัสผ่านต้องมีตัวอักษรพิมพ์เล็กอย่างน้อย 1 ตัว')
    
    if not re.search(r'[0-9]', value):
        raise ValidationError('รหัสผ่านต้องมีตัวเลขอย่างน้อย 1 ตัว')
    
    if not re.search(r'[!@#$%^&*(),.?":{}|<>]', value):
        raise ValidationError('รหัสผ่านต้องมีอักขระพิเศษอย่างน้อย 1 ตัว เช่น !@#$')


def validate_image_size(value):
    """Validator สำหรับขนาดไฟล์รูปภาพ"""
    max_size = 5 * 1024 * 1024  # 5 MB
    if value.size > max_size:
        raise ValidationError(f'ขนาดไฟล์ใหญ่เกินไป (สูงสุด 5 MB)')


# ใช้ validator ใน form
class UserProfileForm(forms.Form):
    phone = forms.CharField(
        validators=[validate_thai_phone],
        label='เบอร์โทรศัพท์'
    )
    password = forms.CharField(
        widget=forms.PasswordInput,
        validators=[validate_strong_password],
        label='รหัสผ่าน'
    )
    avatar = forms.ImageField(
        validators=[validate_image_size],
        required=False,
        label='รูปโปรไฟล์'
    )


# ใช้ validator ใน model field
from django.db import models

class UserProfile(models.Model):
    phone = models.CharField(
        max_length=20,
        validators=[validate_thai_phone]
    )
```

---

## 8. CSRF Protection

CSRF (Cross-Site Request Forgery) เป็นการโจมตีที่ทำให้ user ทำ action โดยไม่ตั้งใจ Django มี CSRF protection built-in

```python
# settings.py - CSRF enabled โดย default
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',  # CSRF middleware
    # ...
]
```

### การใช้ CSRF ใน Template

```html
<!-- ต้องใส่ {% csrf_token %} ใน form ทุก form ที่ใช้ POST -->
<form method="post">
    {% csrf_token %}
    {{ form }}
    <button type="submit">Submit</button>
</form>

<!-- สำหรับ AJAX requests ต้องส่ง CSRF token ใน header -->
<script>
// ดึง CSRF token จาก cookie
function getCookie(name) {
    let cookieValue = null;
    if (document.cookie && document.cookie !== '') {
        const cookies = document.cookie.split(';');
        for (let i = 0; i < cookies.length; i++) {
            const cookie = cookies[i].trim();
            if (cookie.substring(0, name.length + 1) === (name + '=')) {
                cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
                break;
            }
        }
    }
    return cookieValue;
}

const csrftoken = getCookie('csrftoken');

// ใช้ใน fetch
fetch('/api/submit/', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-CSRFToken': csrftoken,  // ส่ง CSRF token
    },
    body: JSON.stringify({data: 'value'})
});
</script>
```

### ยกเว้น CSRF สำหรับ API Views

```python
from django.views.decorators.csrf import csrf_exempt
from django.http import JsonResponse

@csrf_exempt  # ยกเว้น CSRF check - ใช้กับ API ที่มี auth อื่น
def api_webhook(request):
    """Webhook endpoint ที่ไม่ต้องการ CSRF"""
    if request.method == 'POST':
        # ตรวจสอบ signature แทน
        return JsonResponse({'status': 'ok'})
```

---

## 9. File Upload

```python
# forms.py
from django import forms

class DocumentUploadForm(forms.Form):
    """Form สำหรับอัปโหลดเอกสาร"""
    title = forms.CharField(max_length=200, label='ชื่อเอกสาร')
    document = forms.FileField(
        label='เลือกไฟล์',
        help_text='รองรับไฟล์ PDF, DOCX, XLSX ขนาดไม่เกิน 10 MB'
    )
    
    def clean_document(self):
        """Validate ไฟล์"""
        document = self.cleaned_data.get('document')
        
        if document:
            # ตรวจสอบขนาดไฟล์
            max_size = 10 * 1024 * 1024  # 10 MB
            if document.size > max_size:
                raise forms.ValidationError('ขนาดไฟล์เกิน 10 MB')
            
            # ตรวจสอบนามสกุลไฟล์
            allowed_extensions = ['.pdf', '.docx', '.xlsx', '.txt']
            import os
            ext = os.path.splitext(document.name)[1].lower()
            if ext not in allowed_extensions:
                raise forms.ValidationError(
                    f'ไฟล์ประเภทนี้ไม่รองรับ รองรับเฉพาะ: {", ".join(allowed_extensions)}'
                )
        
        return document
```

```python
# models.py
import os
from django.db import models

def document_upload_path(instance, filename):
    """กำหนด path สำหรับเก็บไฟล์"""
    # เก็บในโฟลเดอร์ตาม user ID
    return f'documents/user_{instance.uploader.id}/{filename}'

class Document(models.Model):
    title = models.CharField(max_length=200)
    uploader = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    file = models.FileField(upload_to=document_upload_path)
    uploaded_at = models.DateTimeField(auto_now_add=True)
    
    def __str__(self):
        return self.title
    
    def delete(self, *args, **kwargs):
        """ลบไฟล์เมื่อลบ object"""
        if self.file:
            if os.path.isfile(self.file.path):
                os.remove(self.file.path)
        super().delete(*args, **kwargs)
```

```python
# settings.py
import os

# ที่เก็บไฟล์ที่ upload
MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')
```

```python
# urls.py (main urls.py)
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    # ... URL patterns
]

# เพิ่ม URL สำหรับ media files ใน development
if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

```python
# views.py
from django.shortcuts import render, redirect
from django.contrib.auth.decorators import login_required
from .forms import DocumentUploadForm
from .models import Document

@login_required
def upload_document(request):
    if request.method == 'POST':
        form = DocumentUploadForm(request.POST, request.FILES)
        if form.is_valid():
            doc = Document(
                title=form.cleaned_data['title'],
                uploader=request.user,
                file=form.cleaned_data['document']
            )
            doc.save()
            return redirect('document_list')
    else:
        form = DocumentUploadForm()
    
    return render(request, 'documents/upload.html', {'form': form})
```

```html
<!-- templates/documents/upload.html -->
<!-- สำคัญ: ต้องใส่ enctype="multipart/form-data" สำหรับ file upload -->
<form method="post" enctype="multipart/form-data">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">อัปโหลด</button>
</form>
```

---

## 10. Inline Formsets

ใช้สำหรับจัดการ related objects พร้อมกัน

```python
# models.py
class Order(models.Model):
    customer_name = models.CharField(max_length=200)
    created_at = models.DateTimeField(auto_now_add=True)

class OrderItem(models.Model):
    order = models.ForeignKey(Order, on_delete=models.CASCADE, related_name='items')
    product_name = models.CharField(max_length=200)
    quantity = models.PositiveIntegerField(default=1)
    price = models.DecimalField(max_digits=10, decimal_places=2)
```

```python
# forms.py
from django.forms import inlineformset_factory
from .models import Order, OrderItem

# สร้าง OrderItem formset สำหรับ Order
OrderItemFormSet = inlineformset_factory(
    Order,           # parent model
    OrderItem,       # child model
    fields=['product_name', 'quantity', 'price'],
    extra=3,         # จำนวน empty forms
    can_delete=True  # อนุญาตให้ลบ items ได้
)
```

```python
# views.py
def order_create(request):
    if request.method == 'POST':
        order_form = OrderForm(request.POST)
        formset = OrderItemFormSet(request.POST)
        
        if order_form.is_valid() and formset.is_valid():
            order = order_form.save()
            # บันทึก formset กับ order ที่เพิ่งสร้าง
            items = formset.save(commit=False)
            for item in items:
                item.order = order
                item.save()
            return redirect('order_detail', pk=order.pk)
    else:
        order_form = OrderForm()
        formset = OrderItemFormSet()
    
    return render(request, 'orders/create.html', {
        'order_form': order_form,
        'formset': formset
    })
```

---

## 11. สรุป Part 061

✅ **Django Forms** ใช้จัดการ HTML forms, validation, และ data processing
✅ **Form class** สร้าง form fields เอง พร้อม validation rules
✅ **ModelForm** สร้าง form จาก Model โดยอัตโนมัติ ลดการเขียนโค้ดซ้ำ
✅ **Field-level validation** ใช้ `clean_<fieldname>()` method
✅ **Cross-field validation** ใช้ `clean()` method
✅ **Custom validators** เป็น function ที่ raise ValidationError
✅ **CSRF protection** ป้องกัน Cross-Site Request Forgery โดยใส่ `{% csrf_token %}`
✅ **File upload** ต้องใช้ `request.FILES` และ `enctype="multipart/form-data"`

## ➡️ ถัดไป: Part 062 - Django Templates Advanced

*Part 061/100+ | Python Course - Beginner to World-Class*
