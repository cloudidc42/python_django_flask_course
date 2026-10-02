# Part 58: Django Forms

## เป้าหมายของบทเรียน

- สร้าง Form class และ ModelForm
- เข้าใจ Field types และ validation
- Render forms ใน templates
- ใช้ FormView
- เข้าใจ CSRF protection
- สร้าง Create/Update forms แบบสมบูรณ์

---

## 1. Django Forms คืออะไร?

Django Forms ช่วยจัดการ:
- **Rendering** - แสดง HTML form
- **Validation** - ตรวจสอบข้อมูล
- **Data processing** - แปลงข้อมูลเป็น Python objects

---

## 2. Form Class พื้นฐาน

```python
# blog/forms.py
from django import forms
from django.core.validators import MinLengthValidator, MaxLengthValidator


class ContactForm(forms.Form):
    """
    Form ติดต่อเรา
    สืบทอดจาก forms.Form
    """
    # กำหนด fields
    name = forms.CharField(
        max_length=100,
        label='ชื่อ-นามสกุล',
        widget=forms.TextInput(attrs={
            'class': 'form-control',
            'placeholder': 'กรอกชื่อ-นามสกุล',
        })
    )
    
    email = forms.EmailField(
        label='อีเมล',
        widget=forms.EmailInput(attrs={
            'class': 'form-control',
            'placeholder': 'example@email.com',
        })
    )
    
    subject = forms.CharField(
        max_length=200,
        label='หัวข้อ',
        widget=forms.TextInput(attrs={'class': 'form-control'})
    )
    
    message = forms.CharField(
        label='ข้อความ',
        widget=forms.Textarea(attrs={
            'class': 'form-control',
            'rows': 5,
        }),
        validators=[MinLengthValidator(10, 'ข้อความต้องมีอย่างน้อย 10 ตัวอักษร')]
    )
    
    # Custom validation
    def clean_email(self):
        """
        Validate email field
        method ชื่อ clean_<field_name>
        """
        email = self.cleaned_data.get('email')
        
        # ตัวอย่าง: ตรวจสอบว่าไม่ใช่ email จาก domain บางแห่ง
        blocked_domains = ['spam.com', 'junk.net']
        domain = email.split('@')[-1]
        
        if domain in blocked_domains:
            raise forms.ValidationError(f'ไม่รองรับ email จาก {domain}')
        
        return email  # ต้อง return ค่าเสมอ
    
    def clean(self):
        """
        Validate ข้อมูลหลายๆ fields พร้อมกัน
        """
        cleaned_data = super().clean()
        name = cleaned_data.get('name')
        message = cleaned_data.get('message')
        
        # ตัวอย่าง: ตรวจสอบว่า name ไม่อยู่ใน message (spam detection)
        if name and message and name.lower() in message.lower() and len(message) < 20:
            raise forms.ValidationError('ข้อความดูเหมือน spam')
        
        return cleaned_data
```

---

## 3. Form Fields ทุกประเภท

```python
from django import forms
from django.core.validators import RegexValidator
import datetime


class AllFieldTypesForm(forms.Form):
    # === Text Fields ===
    char_field = forms.CharField(max_length=100)
    text_field = forms.CharField(widget=forms.Textarea)
    email_field = forms.EmailField()
    url_field = forms.URLField()
    slug_field = forms.SlugField()
    
    # === Number Fields ===
    integer_field = forms.IntegerField()
    float_field = forms.FloatField()
    decimal_field = forms.DecimalField(max_digits=10, decimal_places=2)
    
    # Min/Max validation
    age = forms.IntegerField(min_value=0, max_value=150)
    price = forms.DecimalField(
        max_digits=10,
        decimal_places=2,
        min_value=0,
    )
    
    # === Boolean Fields ===
    bool_field = forms.BooleanField()
    null_bool_field = forms.NullBooleanField()
    
    # === Date/Time Fields ===
    date_field = forms.DateField(
        widget=forms.DateInput(attrs={'type': 'date'})
    )
    time_field = forms.TimeField(
        widget=forms.TimeInput(attrs={'type': 'time'})
    )
    datetime_field = forms.DateTimeField(
        widget=forms.DateTimeInput(attrs={'type': 'datetime-local'})
    )
    
    # === Choice Fields ===
    STATUS_CHOICES = [
        ('', '-- เลือกสถานะ --'),
        ('draft', 'แบบร่าง'),
        ('published', 'เผยแพร่'),
        ('archived', 'เก็บเข้าคลัง'),
    ]
    
    choice_field = forms.ChoiceField(choices=STATUS_CHOICES)
    multiple_choice = forms.MultipleChoiceField(
        choices=STATUS_CHOICES,
        widget=forms.CheckboxSelectMultiple
    )
    typed_choice = forms.TypedChoiceField(
        choices=[(1, 'หนึ่ง'), (2, 'สอง'), (3, 'สาม')],
        coerce=int  # แปลงเป็น int
    )
    
    # === File Fields ===
    file_field = forms.FileField()
    image_field = forms.ImageField()
    
    # === Other Fields ===
    ip_field = forms.GenericIPAddressField()
    uuid_field = forms.UUIDField()
    
    # ใช้ regex validator
    phone = forms.CharField(
        validators=[
            RegexValidator(
                regex=r'^\+?1?\d{9,15}$',
                message='กรอกเบอร์โทรศัพท์ที่ถูกต้อง'
            )
        ]
    )
```

### Field Options

```python
class FieldOptionsForm(forms.Form):
    # required - จำเป็นต้องกรอก (default: True)
    required_field = forms.CharField()
    optional_field = forms.CharField(required=False)
    
    # label - ชื่อที่แสดง
    name = forms.CharField(label='ชื่อของคุณ')
    
    # help_text - คำแนะนำ
    password = forms.CharField(
        help_text='รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร'
    )
    
    # initial - ค่าเริ่มต้น
    country = forms.CharField(initial='Thailand')
    
    # widget - HTML element ที่ใช้แสดง
    bio = forms.CharField(
        widget=forms.Textarea(attrs={'rows': 4})
    )
    
    # error_messages - custom error messages
    email = forms.EmailField(
        error_messages={
            'required': 'กรุณากรอกอีเมล',
            'invalid': 'อีเมลไม่ถูกต้อง',
        }
    )
    
    # disabled - ไม่ให้แก้ไข
    username = forms.CharField(disabled=True)
    
    # label_suffix - เปลี่ยน ':' หลัง label
    age = forms.IntegerField(label_suffix=' (ปี):')
```

### Widgets

```python
class WidgetExamplesForm(forms.Form):
    # Text widgets
    text_input = forms.CharField(widget=forms.TextInput)
    password = forms.CharField(widget=forms.PasswordInput)
    textarea = forms.CharField(widget=forms.Textarea(attrs={'rows': 4}))
    hidden = forms.CharField(widget=forms.HiddenInput)
    
    # Number widgets
    number = forms.IntegerField(widget=forms.NumberInput(attrs={'step': 5}))
    
    # Date/Time widgets
    date = forms.DateField(widget=forms.DateInput(attrs={'type': 'date'}))
    time = forms.TimeField(widget=forms.TimeInput(attrs={'type': 'time'}))
    
    # Choice widgets
    dropdown = forms.ChoiceField(
        choices=[('a', 'A'), ('b', 'B')],
        widget=forms.Select
    )
    radio = forms.ChoiceField(
        choices=[('a', 'A'), ('b', 'B')],
        widget=forms.RadioSelect
    )
    checkboxes = forms.MultipleChoiceField(
        choices=[('a', 'A'), ('b', 'B')],
        widget=forms.CheckboxSelectMultiple
    )
    select_multiple = forms.MultipleChoiceField(
        choices=[('a', 'A'), ('b', 'B')],
        widget=forms.SelectMultiple
    )
    
    # File widgets
    file = forms.FileField(widget=forms.FileInput)
    file_multiple = forms.FileField(
        widget=forms.ClearableFileInput(attrs={'multiple': True})
    )
```

---

## 4. ModelForm

ModelForm สร้าง form โดยอัตโนมัติจาก Model

```python
# blog/forms.py
from django import forms
from .models import Post, Comment, Category


class PostForm(forms.ModelForm):
    """
    Form สำหรับสร้าง/แก้ไข Post
    สืบทอดจาก forms.ModelForm
    """
    
    class Meta:
        # model ที่ form นี้อ้างอิง
        model = Post
        
        # fields ที่รวมอยู่ใน form
        # '__all__' = ทุก fields (ไม่แนะนำ)
        fields = ['title', 'slug', 'excerpt', 'content', 'category', 'tags', 
                  'featured_image', 'status', 'is_featured']
        
        # หรือยกเว้น fields
        # exclude = ['author', 'created_at', 'updated_at', 'views_count']
        
        # กำหนด labels
        labels = {
            'title': 'หัวข้อบทความ',
            'slug': 'URL Slug',
            'excerpt': 'บทคัดย่อ',
            'content': 'เนื้อหา',
            'category': 'หมวดหมู่',
            'tags': 'แท็ก',
            'featured_image': 'รูปภาพหลัก',
            'status': 'สถานะ',
            'is_featured': 'บทความแนะนำ',
        }
        
        # กำหนด help_text
        help_texts = {
            'slug': 'URL-friendly version ของหัวข้อ (ใช้ a-z, 0-9, -)',
            'excerpt': 'สรุปสั้นๆ (ไม่เกิน 500 ตัวอักษร)',
            'tags': 'เลือกหลาย tags ได้',
        }
        
        # กำหนด widgets
        widgets = {
            'title': forms.TextInput(attrs={
                'class': 'form-control',
                'placeholder': 'กรอกหัวข้อบทความ',
            }),
            'slug': forms.TextInput(attrs={'class': 'form-control'}),
            'excerpt': forms.Textarea(attrs={
                'class': 'form-control',
                'rows': 3,
            }),
            'content': forms.Textarea(attrs={
                'class': 'form-control',
                'rows': 15,
                'id': 'content-editor',
            }),
            'category': forms.Select(attrs={'class': 'form-select'}),
            'tags': forms.CheckboxSelectMultiple(),
            'featured_image': forms.ClearableFileInput(attrs={'class': 'form-control'}),
            'status': forms.Select(attrs={'class': 'form-select'}),
            'is_featured': forms.CheckboxInput(attrs={'class': 'form-check-input'}),
        }
        
        # กำหนด error_messages
        error_messages = {
            'title': {
                'required': 'กรุณากรอกหัวข้อ',
                'max_length': 'หัวข้อยาวเกินไป',
            }
        }
    
    def clean_slug(self):
        """Validate slug"""
        slug = self.cleaned_data.get('slug')
        
        # ตรวจสอบว่า slug ไม่ซ้ำ
        # instance = self.instance ถ้าเป็น update
        qs = Post.objects.filter(slug=slug)
        if self.instance.pk:
            qs = qs.exclude(pk=self.instance.pk)
        
        if qs.exists():
            raise forms.ValidationError('Slug นี้มีใช้งานอยู่แล้ว')
        
        return slug
    
    def clean_content(self):
        """Validate content"""
        content = self.cleaned_data.get('content', '')
        word_count = len(content.split())
        
        if word_count < 50:
            raise forms.ValidationError(
                f'เนื้อหาน้อยเกินไป (ปัจจุบัน {word_count} คำ, ต้องมีอย่างน้อย 50 คำ)'
            )
        
        return content
    
    def save(self, commit=True):
        """Override save เพื่อ auto-generate slug"""
        instance = super().save(commit=False)
        
        # สร้าง slug ถ้าไม่ได้กรอก
        if not instance.slug:
            from django.utils.text import slugify
            instance.slug = slugify(instance.title)
        
        if commit:
            instance.save()
            self.save_m2m()
        
        return instance
```

### ModelForm สำหรับ Comment

```python
class CommentForm(forms.ModelForm):
    """Form สำหรับเพิ่มความคิดเห็น"""
    
    class Meta:
        model = Comment
        fields = ['content']
        widgets = {
            'content': forms.Textarea(attrs={
                'class': 'form-control',
                'rows': 4,
                'placeholder': 'แสดงความคิดเห็น...',
            })
        }
        labels = {
            'content': 'ความคิดเห็น',
        }
    
    def clean_content(self):
        content = self.cleaned_data.get('content', '').strip()
        if len(content) < 5:
            raise forms.ValidationError('ความคิดเห็นสั้นเกินไป')
        return content
```

---

## 5. Form Rendering

### render as_p, as_table, as_ul

```html
<!-- ใน template -->
<form method="post">
    {% csrf_token %}
    
    <!-- render ทั้ง form เป็น <p> tags -->
    {{ form.as_p }}
    
    <!-- render เป็น <table> -->
    {{ form.as_table }}
    
    <!-- render เป็น <ul> -->
    {{ form.as_ul }}
    
    <button type="submit">บันทึก</button>
</form>
```

### Render แต่ละ Field แยกกัน (แนะนำ)

```html
<form method="post" enctype="multipart/form-data">
    {% csrf_token %}
    
    <!-- วนลูปแสดงทุก fields -->
    {% for field in form %}
    <div class="mb-3">
        <!-- Label -->
        {{ field.label_tag }}
        
        <!-- Input -->
        {{ field }}
        
        <!-- Help text -->
        {% if field.help_text %}
        <div class="form-text">{{ field.help_text }}</div>
        {% endif %}
        
        <!-- Errors -->
        {% if field.errors %}
        <div class="invalid-feedback d-block">
            {% for error in field.errors %}
            {{ error }}
            {% endfor %}
        </div>
        {% endif %}
    </div>
    {% endfor %}
    
    <button type="submit">บันทึก</button>
</form>
```

### Bootstrap Form Template

```html
<!-- templates/blog/post_form.html -->
{% extends 'base.html' %}
{% load crispy_forms_tags %}

{% block title %}{{ title }}{% endblock %}

{% block content %}
<div class="row justify-content-center">
    <div class="col-lg-8">
        <div class="card shadow">
            <div class="card-header">
                <h4 class="mb-0">{{ title }}</h4>
            </div>
            <div class="card-body">
                <form method="post" enctype="multipart/form-data">
                    {% csrf_token %}
                    
                    <!-- Title -->
                    <div class="mb-3">
                        <label for="{{ form.title.id_for_label }}" class="form-label">
                            {{ form.title.label }}
                            {% if form.title.field.required %}
                            <span class="text-danger">*</span>
                            {% endif %}
                        </label>
                        <input type="text" 
                               name="{{ form.title.html_name }}"
                               id="{{ form.title.id_for_label }}"
                               value="{{ form.title.value|default:'' }}"
                               class="form-control {% if form.title.errors %}is-invalid{% endif %}"
                               placeholder="กรอกหัวข้อบทความ">
                        {% for error in form.title.errors %}
                        <div class="invalid-feedback">{{ error }}</div>
                        {% endfor %}
                    </div>
                    
                    <!-- Excerpt -->
                    <div class="mb-3">
                        <label class="form-label">{{ form.excerpt.label }}</label>
                        <textarea name="{{ form.excerpt.html_name }}"
                                  id="{{ form.excerpt.id_for_label }}"
                                  class="form-control {% if form.excerpt.errors %}is-invalid{% endif %}"
                                  rows="3">{{ form.excerpt.value|default:'' }}</textarea>
                        <div class="form-text">{{ form.excerpt.help_text }}</div>
                        {% for error in form.excerpt.errors %}
                        <div class="invalid-feedback">{{ error }}</div>
                        {% endfor %}
                    </div>
                    
                    <!-- Content -->
                    <div class="mb-3">
                        <label class="form-label">
                            {{ form.content.label }}
                            <span class="text-danger">*</span>
                        </label>
                        <textarea name="{{ form.content.html_name }}"
                                  id="{{ form.content.id_for_label }}"
                                  class="form-control {% if form.content.errors %}is-invalid{% endif %}"
                                  rows="12">{{ form.content.value|default:'' }}</textarea>
                        {% for error in form.content.errors %}
                        <div class="invalid-feedback">{{ error }}</div>
                        {% endfor %}
                    </div>
                    
                    <div class="row">
                        <!-- Category -->
                        <div class="col-md-6 mb-3">
                            <label class="form-label">{{ form.category.label }}</label>
                            <select name="{{ form.category.html_name }}"
                                    class="form-select {% if form.category.errors %}is-invalid{% endif %}">
                                <option value="">-- เลือกหมวดหมู่ --</option>
                                {% for choice in form.category.field.choices %}
                                <option value="{{ choice.0 }}"
                                        {% if form.category.value|stringformat:"s" == choice.0|stringformat:"s" %}selected{% endif %}>
                                    {{ choice.1 }}
                                </option>
                                {% endfor %}
                            </select>
                        </div>
                        
                        <!-- Status -->
                        <div class="col-md-6 mb-3">
                            <label class="form-label">{{ form.status.label }}</label>
                            {{ form.status }}
                            {% for error in form.status.errors %}
                            <div class="invalid-feedback d-block">{{ error }}</div>
                            {% endfor %}
                        </div>
                    </div>
                    
                    <!-- Featured Image -->
                    <div class="mb-3">
                        <label class="form-label">{{ form.featured_image.label }}</label>
                        {% if form.instance.featured_image %}
                        <div class="mb-2">
                            <img src="{{ form.instance.featured_image.url }}" 
                                 style="max-height: 150px;" class="img-thumbnail">
                        </div>
                        {% endif %}
                        {{ form.featured_image }}
                    </div>
                    
                    <!-- Tags -->
                    <div class="mb-3">
                        <label class="form-label">{{ form.tags.label }}</label>
                        <div class="row">
                            {% for checkbox in form.tags %}
                            <div class="col-6 col-md-4">
                                <div class="form-check">
                                    {{ checkbox.tag }}
                                    <label class="form-check-label" for="{{ checkbox.id_for_label }}">
                                        {{ checkbox.choice_label }}
                                    </label>
                                </div>
                            </div>
                            {% endfor %}
                        </div>
                    </div>
                    
                    <!-- Is Featured -->
                    <div class="mb-3 form-check">
                        {{ form.is_featured }}
                        <label class="form-check-label" for="{{ form.is_featured.id_for_label }}">
                            {{ form.is_featured.label }}
                        </label>
                    </div>
                    
                    <!-- Non-field errors -->
                    {% if form.non_field_errors %}
                    <div class="alert alert-danger">
                        {% for error in form.non_field_errors %}
                        <p class="mb-0">{{ error }}</p>
                        {% endfor %}
                    </div>
                    {% endif %}
                    
                    <!-- Submit buttons -->
                    <div class="d-flex gap-2">
                        <button type="submit" class="btn btn-primary">
                            <i class="bi bi-save"></i> {{ action }}บทความ
                        </button>
                        <button type="submit" name="save_draft" value="1" class="btn btn-secondary">
                            <i class="bi bi-file-text"></i> บันทึกเป็นร่าง
                        </button>
                        <a href="{% url 'blog:post_list' %}" class="btn btn-outline-secondary">
                            ยกเลิก
                        </a>
                    </div>
                </form>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

---

## 6. CSRF Protection

CSRF (Cross-Site Request Forgery) คือการโจมตีที่ทำให้ผู้ใช้ทำ action โดยไม่ได้ตั้งใจ

```html
<!-- ทุก form ที่ใช้ POST ต้องมี csrf_token -->
<form method="post">
    {% csrf_token %}
    <!-- form fields -->
</form>
```

```python
# สำหรับ AJAX requests
# ต้องส่ง CSRF token ใน header

# JavaScript
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

fetch('/api/endpoint/', {
    method: 'POST',
    headers: {
        'X-CSRFToken': csrftoken,
        'Content-Type': 'application/json',
    },
    body: JSON.stringify(data),
});
```

```python
# view ที่ exempt จาก CSRF (ระวัง!)
from django.views.decorators.csrf import csrf_exempt

@csrf_exempt
def api_endpoint(request):
    """
    สำหรับ external API calls ที่ไม่มี CSRF token
    ควรใช้ authentication อื่นแทน เช่น API key
    """
    pass
```

---

## 7. FormView

```python
# blog/views.py
from django.views.generic import FormView
from django.contrib import messages
from .forms import ContactForm


class ContactView(FormView):
    """
    View สำหรับ contact form
    """
    template_name = 'pages/contact.html'
    form_class = ContactForm
    success_url = '/contact/success/'
    
    def form_valid(self, form):
        """
        ทำงานเมื่อ form valid
        """
        # ดึงข้อมูลที่ validated แล้ว
        name = form.cleaned_data['name']
        email = form.cleaned_data['email']
        subject = form.cleaned_data['subject']
        message = form.cleaned_data['message']
        
        # ส่ง email
        from django.core.mail import send_mail
        send_mail(
            subject=f'[Contact] {subject}',
            message=f'จาก: {name} <{email}>\n\n{message}',
            from_email='noreply@myblog.com',
            recipient_list=['admin@myblog.com'],
        )
        
        messages.success(
            self.request,
            f'ขอบคุณ {name}! เราจะติดต่อกลับเร็วๆ นี้'
        )
        
        return super().form_valid(form)
    
    def form_invalid(self, form):
        """ทำงานเมื่อ form invalid"""
        messages.error(self.request, 'กรุณาตรวจสอบข้อมูลที่กรอก')
        return super().form_invalid(form)
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['page_title'] = 'ติดต่อเรา'
        return context
```

---

## 8. Form กับ File Upload

```python
# forms.py
class UploadProfileForm(forms.ModelForm):
    class Meta:
        model = UserProfile
        fields = ['avatar', 'bio']
        widgets = {
            'avatar': forms.FileInput(attrs={'accept': 'image/*'}),
        }
    
    def clean_avatar(self):
        """Validate uploaded image"""
        avatar = self.cleaned_data.get('avatar')
        
        if avatar:
            # ตรวจสอบขนาดไฟล์ (max 2MB)
            if avatar.size > 2 * 1024 * 1024:
                raise forms.ValidationError('ขนาดรูปภาพต้องไม่เกิน 2MB')
            
            # ตรวจสอบ extension
            import os
            ext = os.path.splitext(avatar.name)[1].lower()
            allowed_extensions = ['.jpg', '.jpeg', '.png', '.gif', '.webp']
            if ext not in allowed_extensions:
                raise forms.ValidationError(
                    f'รองรับเฉพาะ {", ".join(allowed_extensions)}'
                )
        
        return avatar
```

```python
# views.py
def upload_profile(request):
    """View สำหรับ upload รูป profile"""
    if request.method == 'POST':
        # ต้องส่ง request.FILES ด้วย
        form = UploadProfileForm(
            request.POST,
            request.FILES,
            instance=request.user.profile
        )
        if form.is_valid():
            form.save()
            messages.success(request, 'อัปเดต profile สำเร็จ!')
            return redirect('profile')
    else:
        form = UploadProfileForm(instance=request.user.profile)
    
    return render(request, 'accounts/profile_edit.html', {'form': form})
```

```html
<!-- Template ต้องมี enctype -->
<form method="post" enctype="multipart/form-data">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">อัปโหลด</button>
</form>
```

---

## 9. Form ขั้นสูง

### Formsets

```python
from django.forms import formset_factory, modelformset_factory, inlineformset_factory


# Formset = หลาย form instances
OrderItemFormSet = modelformset_factory(
    OrderItem,
    fields=['product', 'quantity'],
    extra=3,  # แสดง 3 forms เปล่าๆ
    can_delete=True,  # ให้ลบได้
)

# Inline Formset (linked to parent)
OrderItemInlineFormSet = inlineformset_factory(
    parent_model=Order,
    model=OrderItem,
    fields=['product', 'quantity', 'price'],
    extra=3,
    can_delete=True,
)


def create_order(request):
    """View สร้าง Order พร้อม items"""
    if request.method == 'POST':
        order_form = OrderForm(request.POST)
        formset = OrderItemInlineFormSet(request.POST, prefix='items')
        
        if order_form.is_valid() and formset.is_valid():
            order = order_form.save(commit=False)
            order.user = request.user
            order.save()
            
            items = formset.save(commit=False)
            for item in items:
                item.order = order
                item.save()
            
            # ลบ items ที่ mark delete
            for item in formset.deleted_objects:
                item.delete()
            
            return redirect('shop:order_detail', order_id=order.pk)
    else:
        order_form = OrderForm()
        formset = OrderItemInlineFormSet(prefix='items')
    
    return render(request, 'shop/order_create.html', {
        'order_form': order_form,
        'formset': formset,
    })
```

---

## 10. ตัวอย่างเต็ม: Registration Form

```python
# accounts/forms.py
from django import forms
from django.contrib.auth.models import User
from django.contrib.auth.password_validation import validate_password
from django.core.exceptions import ValidationError


class RegisterForm(forms.Form):
    """
    Form สมัครสมาชิก
    """
    username = forms.CharField(
        max_length=150,
        label='ชื่อผู้ใช้',
        widget=forms.TextInput(attrs={
            'class': 'form-control',
            'placeholder': 'ชื่อผู้ใช้ (a-z, 0-9, _, @, +, -, .)',
        })
    )
    
    email = forms.EmailField(
        label='อีเมล',
        widget=forms.EmailInput(attrs={
            'class': 'form-control',
            'placeholder': 'example@email.com',
        })
    )
    
    first_name = forms.CharField(
        max_length=150,
        label='ชื่อ',
        widget=forms.TextInput(attrs={'class': 'form-control'})
    )
    
    last_name = forms.CharField(
        max_length=150,
        label='นามสกุล',
        widget=forms.TextInput(attrs={'class': 'form-control'})
    )
    
    password1 = forms.CharField(
        label='รหัสผ่าน',
        widget=forms.PasswordInput(attrs={'class': 'form-control'}),
        help_text='ต้องมีอย่างน้อย 8 ตัวอักษร ไม่ใช่แค่ตัวเลข'
    )
    
    password2 = forms.CharField(
        label='ยืนยันรหัสผ่าน',
        widget=forms.PasswordInput(attrs={'class': 'form-control'})
    )
    
    agree_terms = forms.BooleanField(
        label='ฉันยอมรับเงื่อนไขการใช้งาน',
        widget=forms.CheckboxInput(attrs={'class': 'form-check-input'})
    )
    
    def clean_username(self):
        """ตรวจสอบว่า username ไม่ซ้ำ"""
        username = self.cleaned_data.get('username')
        if User.objects.filter(username=username).exists():
            raise forms.ValidationError('ชื่อผู้ใช้นี้มีผู้ใช้งานแล้ว')
        return username
    
    def clean_email(self):
        """ตรวจสอบว่า email ไม่ซ้ำ"""
        email = self.cleaned_data.get('email')
        if User.objects.filter(email=email).exists():
            raise forms.ValidationError('อีเมลนี้มีผู้ใช้งานแล้ว')
        return email.lower()
    
    def clean_password1(self):
        """Validate password strength"""
        password = self.cleaned_data.get('password1')
        if password:
            try:
                validate_password(password)
            except ValidationError as e:
                raise forms.ValidationError(list(e.messages))
        return password
    
    def clean(self):
        """ตรวจสอบว่า passwords ตรงกัน"""
        cleaned_data = super().clean()
        password1 = cleaned_data.get('password1')
        password2 = cleaned_data.get('password2')
        
        if password1 and password2 and password1 != password2:
            self.add_error('password2', 'รหัสผ่านไม่ตรงกัน')
        
        return cleaned_data
    
    def save(self):
        """สร้าง User จาก form data"""
        data = self.cleaned_data
        user = User.objects.create_user(
            username=data['username'],
            email=data['email'],
            password=data['password1'],
            first_name=data['first_name'],
            last_name=data['last_name'],
        )
        return user
```

```python
# accounts/views.py
from django.shortcuts import render, redirect
from django.contrib.auth import login
from django.contrib import messages
from .forms import RegisterForm


def register(request):
    """View สมัครสมาชิก"""
    if request.user.is_authenticated:
        return redirect('blog:post_list')
    
    if request.method == 'POST':
        form = RegisterForm(request.POST)
        if form.is_valid():
            user = form.save()
            # login อัตโนมัติหลังสมัคร
            login(request, user)
            messages.success(request, f'ยินดีต้อนรับ {user.get_full_name() or user.username}!')
            return redirect('blog:post_list')
        else:
            messages.error(request, 'กรุณาตรวจสอบข้อมูล')
    else:
        form = RegisterForm()
    
    return render(request, 'accounts/register.html', {'form': form})
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Product Form
สร้าง ProductForm สำหรับ e-commerce:
- fields: name, description, price, stock, category, images
- Validation: price > 0, stock >= 0, images < 5MB
- Auto-generate slug จาก name

### แบบฝึกหัดที่ 2: Search Form
สร้าง SearchForm ที่:
- มี fields: keyword, category, min_price, max_price, sort_by
- ไม่มี required fields
- clean() ตรวจสอบว่า min_price <= max_price

### แบบฝึกหัดที่ 3: Multi-step Form
สร้าง checkout form แบบ 3 ขั้นตอน:
- Step 1: ข้อมูลส่วนตัว
- Step 2: ที่อยู่จัดส่ง
- Step 3: การชำระเงิน
- บันทึกข้อมูลแต่ละขั้นใน session

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- Form class และ ModelForm
- Field types และ widgets ทุกประเภท
- Field options (required, label, help_text, widget)
- Custom validation (clean_<field> และ clean)
- Form rendering ใน templates
- CSRF protection
- FormView สำหรับ Class-Based Views
- File upload forms
- Registration form ตัวอย่างเต็ม

---

## บทถัดไป

➡️ **[Part 59: Django Authentication](part-059.md)** - เรียนรู้ระบบ authentication, user management, และ permissions
