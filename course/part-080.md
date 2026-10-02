# Part 080: Flask Forms
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้
- ใช้ Flask-WTF และ WTForms สร้าง forms
- ตรวจสอบข้อมูล (validation) ใน forms
- ป้องกัน CSRF attacks
- สร้าง custom validators
- ใช้ FlaskForm ในหน้า template

---

## 1. Flask-WTF คืออะไร?

Flask-WTF เป็น extension ที่รวม WTForms เข้ากับ Flask เพิ่มความสามารถ:
- **CSRF Protection**: ป้องกัน Cross-Site Request Forgery
- **File Uploads**: รองรับการอัปโหลดไฟล์
- **ReCAPTCHA**: รองรับ Google ReCAPTCHA

### ติดตั้ง
```bash
pip install flask-wtf
pip install email-validator  # สำหรับ validate email
```

### Configuration
```python
# app.py

from flask import Flask
from flask_wtf.csrf import CSRFProtect

app = Flask(__name__)

# SECRET_KEY จำเป็นสำหรับ CSRF token
app.config['SECRET_KEY'] = 'your-secret-key-here-change-in-production'

# เปิดใช้ CSRF protection ทั่วทั้ง app
csrf = CSRFProtect(app)

# กำหนดเวลา expire ของ CSRF token (วินาที)
app.config['WTF_CSRF_TIME_LIMIT'] = 3600  # 1 ชั่วโมง
```

---

## 2. สร้าง Form ด้วย FlaskForm

### Form พื้นฐาน
```python
# forms.py

from flask_wtf import FlaskForm
from wtforms import (
    StringField, PasswordField, EmailField,
    TextAreaField, BooleanField, SelectField,
    IntegerField, FloatField, DateField,
    SubmitField
)
from wtforms.validators import (
    DataRequired, Email, Length, EqualTo,
    NumberRange, Optional, URL, Regexp
)


class LoginForm(FlaskForm):
    """Form สำหรับ login"""
    
    # StringField — input text ทั่วไป
    username = StringField(
        'ชื่อผู้ใช้',                    # label
        validators=[
            DataRequired(message='กรุณาใส่ชื่อผู้ใช้'),
            Length(min=3, max=80, message='ชื่อผู้ใช้ต้องมี 3-80 ตัวอักษร')
        ]
    )
    
    # PasswordField — input password
    password = PasswordField(
        'รหัสผ่าน',
        validators=[
            DataRequired(message='กรุณาใส่รหัสผ่าน'),
            Length(min=8, message='รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร')
        ]
    )
    
    # BooleanField — checkbox
    remember_me = BooleanField('จดจำฉัน')
    
    # SubmitField — submit button
    submit = SubmitField('เข้าสู่ระบบ')


class RegisterForm(FlaskForm):
    """Form สำหรับสมัครสมาชิก"""
    
    username = StringField(
        'ชื่อผู้ใช้',
        validators=[
            DataRequired(message='กรุณาใส่ชื่อผู้ใช้'),
            Length(min=3, max=80),
            # Regexp — ตรวจสอบรูปแบบ
            Regexp(
                r'^[a-zA-Z0-9_]+$',
                message='ชื่อผู้ใช้ใช้ได้เฉพาะตัวอักษร ตัวเลข และ _'
            )
        ]
    )
    
    # EmailField — ตรวจสอบรูปแบบ email
    email = EmailField(
        'อีเมล',
        validators=[
            DataRequired(message='กรุณาใส่อีเมล'),
            Email(message='รูปแบบอีเมลไม่ถูกต้อง')
        ]
    )
    
    password = PasswordField(
        'รหัสผ่าน',
        validators=[
            DataRequired(),
            Length(min=8, max=128)
        ]
    )
    
    # EqualTo — ตรวจสอบว่าเท่ากับ field อื่น
    confirm_password = PasswordField(
        'ยืนยันรหัสผ่าน',
        validators=[
            DataRequired(message='กรุณายืนยันรหัสผ่าน'),
            EqualTo('password', message='รหัสผ่านไม่ตรงกัน')
        ]
    )
    
    submit = SubmitField('สมัครสมาชิก')
```

---

## 3. WTForms Validators

### Validators ที่ใช้บ่อย
```python
from wtforms.validators import (
    DataRequired,    # ห้ามเป็นค่าว่าง
    Optional,        # อนุญาตให้ว่างได้
    Email,           # ตรวจสอบรูปแบบ email
    Length,          # ตรวจสอบความยาว
    NumberRange,     # ตรวจสอบช่วงตัวเลข
    EqualTo,         # ต้องเท่ากับ field อื่น
    URL,             # ตรวจสอบรูปแบบ URL
    Regexp,          # ตรวจสอบด้วย regex
    InputRequired,   # ต้องมีข้อมูล (ต่างจาก DataRequired ตรงที่ตรวจ input โดยตรง)
    AnyOf,           # ต้องเป็นหนึ่งในค่าที่กำหนด
    NoneOf,          # ต้องไม่เป็นค่าที่กำหนด
)


class ProductForm(FlaskForm):
    """ตัวอย่าง Form สำหรับสินค้า"""
    
    name = StringField(
        'ชื่อสินค้า',
        validators=[
            DataRequired(message='กรุณาใส่ชื่อสินค้า'),
            Length(min=2, max=200, message='ชื่อสินค้าต้องมี 2-200 ตัวอักษร')
        ]
    )
    
    description = TextAreaField(
        'คำอธิบาย',
        validators=[
            Optional(),  # ไม่จำเป็นต้องใส่
            Length(max=2000)
        ]
    )
    
    price = FloatField(
        'ราคา (บาท)',
        validators=[
            DataRequired(message='กรุณาใส่ราคา'),
            NumberRange(min=0.01, max=9999999, message='ราคาต้องอยู่ระหว่าง 0.01 - 9,999,999')
        ]
    )
    
    stock = IntegerField(
        'จำนวนในสต็อก',
        validators=[
            DataRequired(),
            NumberRange(min=0, message='จำนวนต้องไม่ต่ำกว่า 0')
        ]
    )
    
    category = SelectField(
        'หมวดหมู่',
        choices=[
            ('', '-- เลือกหมวดหมู่ --'),
            ('electronics', 'อิเล็กทรอนิกส์'),
            ('clothing', 'เสื้อผ้า'),
            ('food', 'อาหาร'),
            ('books', 'หนังสือ'),
        ],
        validators=[DataRequired(message='กรุณาเลือกหมวดหมู่')]
    )
    
    website = StringField(
        'เว็บไซต์',
        validators=[
            Optional(),
            URL(message='รูปแบบ URL ไม่ถูกต้อง')
        ]
    )
    
    is_available = BooleanField('พร้อมขาย', default=True)
    
    launch_date = DateField(
        'วันที่เริ่มขาย',
        validators=[Optional()],
        format='%Y-%m-%d'
    )
    
    submit = SubmitField('บันทึกสินค้า')
```

---

## 4. Custom Validators

### Validator แบบ Function
```python
# validators.py

from wtforms.validators import ValidationError
from models import User


def username_unique(form, field):
    """ตรวจสอบว่า username ไม่ซ้ำในฐานข้อมูล"""
    user = User.query.filter_by(username=field.data).first()
    if user:
        raise ValidationError('ชื่อผู้ใช้นี้มีคนใช้แล้ว กรุณาเลือกชื่ออื่น')


def email_unique(form, field):
    """ตรวจสอบว่า email ไม่ซ้ำในฐานข้อมูล"""
    user = User.query.filter_by(email=field.data.lower()).first()
    if user:
        raise ValidationError('อีเมลนี้มีคนใช้แล้ว กรุณาใช้อีเมลอื่น')


def strong_password(form, field):
    """ตรวจสอบว่า password แข็งแรงพอ"""
    password = field.data
    errors = []
    
    if not any(c.isupper() for c in password):
        errors.append('ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว')
    
    if not any(c.islower() for c in password):
        errors.append('ต้องมีตัวพิมพ์เล็กอย่างน้อย 1 ตัว')
    
    if not any(c.isdigit() for c in password):
        errors.append('ต้องมีตัวเลขอย่างน้อย 1 ตัว')
    
    special = set('!@#$%^&*()_+-=[]{}|;:,.<>?')
    if not any(c in special for c in password):
        errors.append('ต้องมีอักขระพิเศษอย่างน้อย 1 ตัว')
    
    if errors:
        raise ValidationError('รหัสผ่านไม่แข็งแรง: ' + ', '.join(errors))
```

### Validator แบบ Method ใน Form
```python
# forms.py

from flask_wtf import FlaskForm
from wtforms import StringField, PasswordField, SubmitField
from wtforms.validators import DataRequired, Length, Email
from models import User


class RegisterForm(FlaskForm):
    """Form พร้อม custom validators"""
    
    username = StringField('ชื่อผู้ใช้', validators=[
        DataRequired(), Length(min=3, max=80)
    ])
    
    email = StringField('อีเมล', validators=[
        DataRequired(), Email()
    ])
    
    password = PasswordField('รหัสผ่าน', validators=[
        DataRequired(), Length(min=8)
    ])
    
    submit = SubmitField('สมัครสมาชิก')
    
    # Custom validator: validate_<fieldname>
    # Flask-WTF จะเรียก method นี้อัตโนมัติเมื่อ validate() field นั้น
    
    def validate_username(self, username):
        """ตรวจสอบ username ไม่ซ้ำ"""
        user = User.query.filter_by(username=username.data).first()
        if user:
            raise ValidationError('ชื่อผู้ใช้นี้มีคนใช้แล้ว')
    
    def validate_email(self, email):
        """ตรวจสอบ email ไม่ซ้ำ"""
        user = User.query.filter_by(email=email.data.lower()).first()
        if user:
            raise ValidationError('อีเมลนี้มีคนใช้แล้ว')
    
    def validate_password(self, password):
        """ตรวจสอบความแข็งแรงของ password"""
        if not any(c.isupper() for c in password.data):
            raise ValidationError('รหัสผ่านต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว')
```

### Validator แบบ Class
```python
# validators.py — Reusable validator classes

from wtforms.validators import ValidationError


class ThaiPhone:
    """ตรวจสอบเบอร์โทรศัพท์ไทย"""
    
    def __init__(self, message=None):
        if not message:
            message = 'เบอร์โทรศัพท์ไม่ถูกต้อง (ต้องเป็น 10 หลักขึ้นต้นด้วย 0)'
        self.message = message
    
    def __call__(self, form, field):
        import re
        phone = field.data
        if phone:
            # ลบ space และ dash
            phone = phone.replace(' ', '').replace('-', '')
            if not re.match(r'^0[0-9]{9}$', phone):
                raise ValidationError(self.message)
            field.data = phone  # เก็บ format ที่ clean แล้ว


class MinWords:
    """ตรวจสอบจำนวนคำขั้นต่ำ"""
    
    def __init__(self, min_count, message=None):
        self.min_count = min_count
        if not message:
            message = f'ต้องมีอย่างน้อย {min_count} คำ'
        self.message = message
    
    def __call__(self, form, field):
        if field.data:
            word_count = len(field.data.split())
            if word_count < self.min_count:
                raise ValidationError(
                    f'ปัจจุบันมี {word_count} คำ ต้องมีอย่างน้อย {self.min_count} คำ'
                )


# ใช้ custom validators
class ArticleForm(FlaskForm):
    title = StringField('หัวเรื่อง', validators=[DataRequired()])
    phone = StringField('เบอร์โทร', validators=[Optional(), ThaiPhone()])
    content = TextAreaField('เนื้อหา', validators=[
        DataRequired(),
        MinWords(100)  # ต้องมีอย่างน้อย 100 คำ
    ])
    submit = SubmitField('บันทึก')
```

---

## 5. CSRF Protection

### CSRF คืออะไร?
Cross-Site Request Forgery คือการโจมตีที่ผู้ไม่หวังดีหลอกให้ผู้ใช้ส่ง request โดยไม่รู้ตัว

### วิธีทำงานของ CSRF Token
1. Flask สร้าง token แบบ random เมื่อผู้ใช้เปิดหน้า
2. Token ถูกฝังใน form เป็น hidden field
3. เมื่อ submit form จะส่ง token กลับมา
4. Flask ตรวจสอบว่า token ถูกต้องก่อนประมวลผล

### ใช้ CSRF ใน Form (FlaskForm ทำให้อัตโนมัติ)
```python
# forms.py — FlaskForm มี CSRF ในตัว
class ContactForm(FlaskForm):
    name = StringField('ชื่อ', validators=[DataRequired()])
    email = EmailField('อีเมล', validators=[DataRequired(), Email()])
    message = TextAreaField('ข้อความ', validators=[DataRequired()])
    submit = SubmitField('ส่งข้อความ')
```

```html
<!-- template: contact.html -->
<form method="POST" action="/contact">
    <!-- Flask-WTF เพิ่ม CSRF token อัตโนมัติเมื่อใช้ form.hidden_tag() -->
    {{ form.hidden_tag() }}
    
    <div>
        {{ form.name.label }}
        {{ form.name(class="form-control") }}
        {% if form.name.errors %}
            {% for error in form.name.errors %}
                <span class="error">{{ error }}</span>
            {% endfor %}
        {% endif %}
    </div>
    
    {{ form.submit(class="btn btn-primary") }}
</form>
```

### ปิด CSRF สำหรับ API endpoints
```python
from flask_wtf.csrf import CSRFProtect, exempt_csrf

csrf = CSRFProtect(app)

# ปิด CSRF สำหรับ route เดียว
@app.route('/api/webhook', methods=['POST'])
@csrf.exempt
def webhook():
    """API endpoint ที่ไม่ต้องการ CSRF"""
    return jsonify({'status': 'ok'})


# หรือปิดทั้ง Blueprint
from flask import Blueprint
api_bp = Blueprint('api', __name__)
csrf.exempt(api_bp)
```

---

## 6. ใช้ Form ใน Route

```python
# routes.py

from flask import Flask, render_template, redirect, url_for, flash
from forms import RegisterForm, LoginForm, ContactForm

app = Flask(__name__)
app.config['SECRET_KEY'] = 'your-secret-key'


@app.route('/register', methods=['GET', 'POST'])
def register():
    """หน้าสมัครสมาชิก"""
    form = RegisterForm()
    
    # validate_on_submit() = ตรวจสอบว่า:
    # 1. เป็น POST request
    # 2. CSRF token ถูกต้อง
    # 3. ข้อมูลในทุก field ผ่าน validators
    if form.validate_on_submit():
        # ดึงข้อมูลจาก form
        username = form.username.data
        email = form.email.data
        password = form.password.data
        
        # สร้าง user (ตัวอย่าง)
        # user = User.create(username, email, password)
        
        flash(f'สมัครสมาชิกสำเร็จ! ยินดีต้อนรับ {username}', 'success')
        return redirect(url_for('login'))
    
    # แสดง form (GET request หรือ validation fail)
    return render_template('register.html', form=form)


@app.route('/login', methods=['GET', 'POST'])
def login():
    """หน้าเข้าสู่ระบบ"""
    form = LoginForm()
    
    if form.validate_on_submit():
        username = form.username.data
        password = form.password.data
        remember = form.remember_me.data
        
        # ตรวจสอบ credential (ตัวอย่าง)
        # user = User.get_by_username(username)
        # if user and user.check_password(password):
        #     login_user(user, remember=remember)
        #     return redirect(url_for('dashboard'))
        
        # ถ้า login ล้มเหลว
        form.username.errors.append('ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง')
    
    return render_template('login.html', form=form)


@app.route('/contact', methods=['GET', 'POST'])
def contact():
    """หน้าติดต่อ"""
    form = ContactForm()
    
    if form.validate_on_submit():
        # ส่งอีเมล (ตัวอย่าง)
        # send_email(
        #     to='admin@example.com',
        #     subject=f'ข้อความจาก {form.name.data}',
        #     body=form.message.data
        # )
        flash('ส่งข้อความสำเร็จ! เราจะติดต่อกลับโดยเร็ว', 'success')
        return redirect(url_for('contact'))
    
    return render_template('contact.html', form=form)
```

---

## 7. Templates สำหรับ Forms

### Template พื้นฐาน
```html
<!-- templates/register.html -->
{% extends "base.html" %}

{% block content %}
<div class="container">
    <h1>สมัครสมาชิก</h1>
    
    <form method="POST" novalidate>
        {{ form.hidden_tag() }}  {# CSRF token #}
        
        {# Username field #}
        <div class="form-group {% if form.username.errors %}has-error{% endif %}">
            {{ form.username.label(class="form-label") }}
            {{ form.username(class="form-control", placeholder="กรอกชื่อผู้ใช้") }}
            {% for error in form.username.errors %}
                <div class="text-danger">{{ error }}</div>
            {% endfor %}
        </div>
        
        {# Email field #}
        <div class="form-group {% if form.email.errors %}has-error{% endif %}">
            {{ form.email.label(class="form-label") }}
            {{ form.email(class="form-control", placeholder="กรอกอีเมล") }}
            {% for error in form.email.errors %}
                <div class="text-danger">{{ error }}</div>
            {% endfor %}
        </div>
        
        {# Password field #}
        <div class="form-group">
            {{ form.password.label(class="form-label") }}
            {{ form.password(class="form-control") }}
            {% if form.password.errors %}
                {% for error in form.password.errors %}
                    <div class="text-danger">{{ error }}</div>
                {% endfor %}
            {% endif %}
        </div>
        
        {# Submit button #}
        {{ form.submit(class="btn btn-primary btn-block") }}
    </form>
    
    <p class="mt-3">มีบัญชีแล้ว? <a href="{{ url_for('login') }}">เข้าสู่ระบบ</a></p>
</div>
{% endblock %}
```

### Macro สำหรับ render form fields
```html
<!-- templates/macros/forms.html -->
{% macro render_field(field, **kwargs) %}
<div class="form-group mb-3">
    {{ field.label(class="form-label fw-bold") }}
    {{ field(class="form-control" + (" is-invalid" if field.errors else ""), **kwargs) }}
    {% if field.description %}
        <div class="form-text text-muted">{{ field.description }}</div>
    {% endif %}
    {% for error in field.errors %}
        <div class="invalid-feedback d-block">{{ error }}</div>
    {% endfor %}
</div>
{% endmacro %}

{% macro render_checkbox(field) %}
<div class="form-check mb-3">
    {{ field(class="form-check-input") }}
    {{ field.label(class="form-check-label") }}
    {% for error in field.errors %}
        <div class="text-danger small">{{ error }}</div>
    {% endfor %}
</div>
{% endmacro %}
```

### ใช้ Macro
```html
<!-- templates/register.html -->
{% extends "base.html" %}
{% from "macros/forms.html" import render_field, render_checkbox %}

{% block content %}
<div class="container" style="max-width: 500px;">
    <h2>สมัครสมาชิก</h2>
    
    <form method="POST" novalidate>
        {{ form.hidden_tag() }}
        
        {{ render_field(form.username, placeholder="กรอกชื่อผู้ใช้") }}
        {{ render_field(form.email, placeholder="กรอกอีเมล") }}
        {{ render_field(form.password) }}
        {{ render_field(form.confirm_password) }}
        
        <div class="d-grid gap-2">
            {{ form.submit(class="btn btn-primary") }}
        </div>
    </form>
</div>
{% endblock %}
```

---

## 8. File Upload Form

```python
# forms.py — File Upload Form

from flask_wtf import FlaskForm
from flask_wtf.file import FileField, FileRequired, FileAllowed
from wtforms import StringField, SubmitField
from wtforms.validators import DataRequired


class ProfileForm(FlaskForm):
    """Form สำหรับแก้ไขโปรไฟล์พร้อม upload รูป"""
    
    username = StringField('ชื่อผู้ใช้', validators=[DataRequired()])
    
    # FileField สำหรับอัปโหลดไฟล์
    avatar = FileField(
        'รูปโปรไฟล์',
        validators=[
            # FileRequired() — บังคับต้องอัปโหลด
            # FileAllowed — จำกัดประเภทไฟล์
            FileAllowed(['jpg', 'jpeg', 'png', 'gif'], 'ใช้ได้เฉพาะไฟล์รูปภาพ!')
        ]
    )
    
    submit = SubmitField('บันทึกโปรไฟล์')
```

```python
# routes.py — Handle File Upload

import os
from flask import current_app
from werkzeug.utils import secure_filename


@app.route('/profile/edit', methods=['GET', 'POST'])
def edit_profile():
    form = ProfileForm()
    
    if form.validate_on_submit():
        username = form.username.data
        
        # Handle file upload
        if form.avatar.data:
            file = form.avatar.data
            filename = secure_filename(file.filename)
            
            # สร้างชื่อไฟล์ unique
            import uuid
            ext = filename.rsplit('.', 1)[1].lower()
            unique_filename = f"{uuid.uuid4().hex}.{ext}"
            
            # บันทึกไฟล์
            upload_folder = os.path.join(current_app.static_folder, 'uploads')
            os.makedirs(upload_folder, exist_ok=True)
            file.save(os.path.join(upload_folder, unique_filename))
            
            # บันทึก path ลงฐานข้อมูล
            avatar_url = f'/static/uploads/{unique_filename}'
        
        flash('บันทึกโปรไฟล์สำเร็จ', 'success')
        return redirect(url_for('profile'))
    
    return render_template('edit_profile.html', form=form)
```

---

## 9. Dynamic Form Fields

```python
# forms.py — Form ที่เปลี่ยน choices แบบ dynamic

from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()


class PostForm(FlaskForm):
    """Form สำหรับสร้างบทความ"""
    
    title = StringField('หัวเรื่อง', validators=[DataRequired()])
    content = TextAreaField('เนื้อหา', validators=[DataRequired()])
    
    # Category จาก database
    category_id = SelectField('หมวดหมู่', coerce=int, validators=[DataRequired()])
    
    submit = SubmitField('บันทึก')
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        # โหลด choices จาก database
        from models import Category
        self.category_id.choices = [
            (c.id, c.name) 
            for c in Category.query.order_by(Category.name).all()
        ]
        # เพิ่ม option ว่างที่ต้น
        self.category_id.choices.insert(0, (0, '-- เลือกหมวดหมู่ --'))
```

---

## 10. Form Validation ใน API

```python
# api_forms.py — ใช้ WTForms สำหรับ validate JSON API

from wtforms import Form, StringField, IntegerField, FloatField
from wtforms.validators import DataRequired, Length, NumberRange


class ProductAPIForm(Form):
    """Form สำหรับ validate JSON API (ไม่ใช้ FlaskForm เพราะไม่มี CSRF)"""
    
    name = StringField(validators=[DataRequired(), Length(min=2, max=200)])
    price = FloatField(validators=[DataRequired(), NumberRange(min=0)])
    stock = IntegerField(validators=[DataRequired(), NumberRange(min=0)])


@app.route('/api/products', methods=['POST'])
def create_product():
    """API: สร้างสินค้าใหม่"""
    data = request.get_json()
    
    if not data:
        return jsonify({'error': 'ต้องส่งข้อมูลเป็น JSON'}), 400
    
    # สร้าง form จาก JSON data
    from werkzeug.datastructures import MultiDict
    form_data = MultiDict(data)
    form = ProductAPIForm(form_data)
    
    if not form.validate():
        return jsonify({
            'error': 'ข้อมูลไม่ถูกต้อง',
            'details': form.errors
        }), 422
    
    # สร้างสินค้า (ตัวอย่าง)
    product = {
        'id': 1,
        'name': form.name.data,
        'price': form.price.data,
        'stock': form.stock.data
    }
    
    return jsonify(product), 201
```

---

## 11. ตัวอย่าง Contact Form สมบูรณ์

```python
# app.py

from flask import Flask, render_template, redirect, url_for, flash
from flask_wtf import FlaskForm
from wtforms import StringField, EmailField, TextAreaField, SelectField, SubmitField
from wtforms.validators import DataRequired, Email, Length

app = Flask(__name__)
app.config['SECRET_KEY'] = 'secret-key-change-this'


class ContactForm(FlaskForm):
    name = StringField('ชื่อ-นามสกุล', validators=[
        DataRequired(message='กรุณาใส่ชื่อ'),
        Length(min=2, max=100)
    ])
    email = EmailField('อีเมล', validators=[
        DataRequired(message='กรุณาใส่อีเมล'),
        Email(message='อีเมลไม่ถูกต้อง')
    ])
    subject = SelectField('หัวข้อ', choices=[
        ('general', 'สอบถามทั่วไป'),
        ('support', 'ขอความช่วยเหลือ'),
        ('feedback', 'ให้ feedback'),
        ('business', 'ธุรกิจ'),
    ])
    message = TextAreaField('ข้อความ', validators=[
        DataRequired(message='กรุณาใส่ข้อความ'),
        Length(min=20, max=2000, message='ข้อความต้องมี 20-2000 ตัวอักษร')
    ])
    submit = SubmitField('ส่งข้อความ')


@app.route('/contact', methods=['GET', 'POST'])
def contact():
    form = ContactForm()
    
    if form.validate_on_submit():
        # บันทึกหรือส่งอีเมล (ตัวอย่าง)
        print(f"From: {form.name.data} <{form.email.data}>")
        print(f"Subject: {form.subject.data}")
        print(f"Message: {form.message.data}")
        
        flash('ขอบคุณ! เราได้รับข้อความของคุณแล้ว จะติดต่อกลับเร็วๆ นี้', 'success')
        return redirect(url_for('contact'))
    
    return render_template('contact.html', form=form)


if __name__ == '__main__':
    app.run(debug=True)
```

---

## 12. สรุป Part 080

✅ **Flask-WTF** รวม WTForms เข้ากับ Flask พร้อม CSRF protection  
✅ **FlaskForm** เป็น base class สำหรับสร้าง form  
✅ **Field types**: StringField, PasswordField, EmailField, TextAreaField, SelectField, BooleanField, FileField  
✅ **Validators**: DataRequired, Email, Length, EqualTo, NumberRange, Regexp, URL  
✅ **Custom validators**: สร้างเป็น function, method ใน class, หรือ validator class  
✅ **CSRF protection**: `form.hidden_tag()` ใน template เพิ่ม token อัตโนมัติ  
✅ **File Upload**: FileField + FileAllowed validator  

---

## ➡️ ถัดไป: Part 081 - Flask Authentication

*Part 080/100+ | Python Course - Beginner to World-Class*
