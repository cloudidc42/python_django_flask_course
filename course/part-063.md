# Part 063: Django Authentication

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Django User model และ authentication system
- ทำ login/logout สำหรับเว็บไซต์
- ใช้ permissions และ groups จัดการสิทธิ์
- ใช้ `@login_required` decorator ป้องกัน views
- สร้าง AbstractUser และ Custom User Model

---

## 1. Django Authentication System

Django มี authentication system built-in ที่รองรับ:
- **Users**: บัญชีผู้ใช้
- **Permissions**: สิทธิ์การกระทำ
- **Groups**: กลุ่มผู้ใช้
- **Sessions**: การจัดการ session

```python
# settings.py - ต้องมี apps เหล่านี้
INSTALLED_APPS = [
    'django.contrib.auth',           # Authentication framework
    'django.contrib.contenttypes',   # Content types framework (ต้องใช้กับ auth)
    'django.contrib.sessions',       # Session framework
    # ...
]

MIDDLEWARE = [
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',  # จัดการ request.user
    # ...
]
```

---

## 2. User Model

```python
# User model มี fields เหล่านี้ built-in:
from django.contrib.auth.models import User

# Fields หลัก:
# username        - ชื่อผู้ใช้ (unique)
# email           - อีเมล
# password        - รหัสผ่าน (hashed โดยอัตโนมัติ)
# first_name      - ชื่อ
# last_name       - นามสกุล
# is_active       - บัญชีเปิดใช้งานอยู่หรือไม่
# is_staff        - เข้า admin ได้หรือไม่
# is_superuser    - superuser มีสิทธิ์ทุกอย่าง
# date_joined     - วันที่สมัคร
# last_login      - เข้าสู่ระบบล่าสุด

# สร้าง user
user = User.objects.create_user(
    username='john',
    email='john@example.com',
    password='SecurePass123!'
)

# สร้าง superuser
superuser = User.objects.create_superuser(
    username='admin',
    email='admin@example.com',
    password='AdminPass123!'
)

# ตรวจสอบ password
user.check_password('SecurePass123!')  # True

# เปลี่ยน password
user.set_password('NewPass456!')
user.save()

# ดึง full name
full_name = user.get_full_name()  # "John Doe"

# ตรวจสอบ permission
user.has_perm('myapp.add_article')
user.has_perms(['myapp.add_article', 'myapp.change_article'])
```

---

## 3. Login / Logout Views

### วิธีที่ 1: ใช้ Django Built-in Views

```python
# urls.py
from django.contrib.auth import views as auth_views
from django.urls import path

urlpatterns = [
    # Django built-in auth views
    path('login/', auth_views.LoginView.as_view(
        template_name='auth/login.html',
        redirect_authenticated_user=True,
    ), name='login'),
    
    path('logout/', auth_views.LogoutView.as_view(
        next_page='home'  # redirect ไปหน้าไหนหลัง logout
    ), name='logout'),
    
    # Password change
    path('password-change/', auth_views.PasswordChangeView.as_view(
        template_name='auth/password_change.html',
        success_url='/password-change/done/'
    ), name='password_change'),
    
    path('password-change/done/', auth_views.PasswordChangeDoneView.as_view(
        template_name='auth/password_change_done.html'
    ), name='password_change_done'),
    
    # Password reset
    path('password-reset/', auth_views.PasswordResetView.as_view(
        template_name='auth/password_reset.html',
        email_template_name='auth/password_reset_email.html',
    ), name='password_reset'),
    
    path('password-reset/done/', auth_views.PasswordResetDoneView.as_view(
        template_name='auth/password_reset_done.html'
    ), name='password_reset_done'),
    
    path('password-reset/<uidb64>/<token>/', auth_views.PasswordResetConfirmView.as_view(
        template_name='auth/password_reset_confirm.html'
    ), name='password_reset_confirm'),
    
    path('password-reset/complete/', auth_views.PasswordResetCompleteView.as_view(
        template_name='auth/password_reset_complete.html'
    ), name='password_reset_complete'),
]
```

```python
# settings.py
LOGIN_URL = '/login/'              # URL สำหรับ login
LOGIN_REDIRECT_URL = '/dashboard/' # redirect ไปหน้าไหนหลัง login
LOGOUT_REDIRECT_URL = '/'          # redirect ไปหน้าไหนหลัง logout
```

```html
<!-- templates/auth/login.html -->
{% extends 'base.html' %}

{% block title %}เข้าสู่ระบบ{% endblock %}

{% block content %}
<div class="row justify-content-center">
    <div class="col-md-4">
        <div class="card shadow">
            <div class="card-body">
                <h4 class="card-title text-center mb-4">เข้าสู่ระบบ</h4>
                
                <form method="post">
                    {% csrf_token %}
                    
                    {% if form.errors %}
                    <div class="alert alert-danger">
                        ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง
                    </div>
                    {% endif %}
                    
                    <div class="mb-3">
                        <label class="form-label">ชื่อผู้ใช้</label>
                        {{ form.username }}
                    </div>
                    
                    <div class="mb-3">
                        <label class="form-label">รหัสผ่าน</label>
                        {{ form.password }}
                    </div>
                    
                    <!-- Redirect ไปหน้าที่ต้องการหลัง login -->
                    <input type="hidden" name="next" value="{{ next }}">
                    
                    <button type="submit" class="btn btn-primary w-100">
                        เข้าสู่ระบบ
                    </button>
                </form>
                
                <div class="text-center mt-3">
                    <a href="{% url 'password_reset' %}">ลืมรหัสผ่าน?</a>
                    <br>
                    <a href="{% url 'register' %}">สมัครสมาชิก</a>
                </div>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

### วิธีที่ 2: Custom Login View

```python
# views.py
from django.shortcuts import render, redirect
from django.contrib.auth import authenticate, login, logout
from django.contrib.auth.forms import AuthenticationForm
from django.contrib import messages

def login_view(request):
    """Custom login view"""
    # ถ้า login แล้ว redirect ไป dashboard
    if request.user.is_authenticated:
        return redirect('dashboard')
    
    if request.method == 'POST':
        form = AuthenticationForm(request, data=request.POST)
        if form.is_valid():
            username = form.cleaned_data.get('username')
            password = form.cleaned_data.get('password')
            
            # ตรวจสอบ credentials
            user = authenticate(request, username=username, password=password)
            
            if user is not None:
                login(request, user)  # สร้าง session
                messages.success(request, f'ยินดีต้อนรับ {user.get_full_name() or user.username}!')
                
                # Redirect ไปหน้าที่ต้องการ หรือ dashboard
                next_url = request.GET.get('next', 'dashboard')
                return redirect(next_url)
            else:
                messages.error(request, 'ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง')
        else:
            messages.error(request, 'ข้อมูลที่กรอกไม่ถูกต้อง')
    else:
        form = AuthenticationForm()
    
    return render(request, 'auth/login.html', {'form': form})


def logout_view(request):
    """Custom logout view"""
    logout(request)
    messages.info(request, 'ออกจากระบบเรียบร้อยแล้ว')
    return redirect('home')


def register_view(request):
    """View สมัครสมาชิก"""
    if request.user.is_authenticated:
        return redirect('dashboard')
    
    if request.method == 'POST':
        form = UserRegistrationForm(request.POST)
        if form.is_valid():
            user = form.save()
            # Login ทันทีหลังสมัคร
            login(request, user)
            messages.success(request, 'สมัครสมาชิกสำเร็จ! ยินดีต้อนรับ')
            return redirect('dashboard')
    else:
        form = UserRegistrationForm()
    
    return render(request, 'auth/register.html', {'form': form})
```

```python
# forms.py
from django import forms
from django.contrib.auth.models import User
from django.contrib.auth.forms import UserCreationForm

class UserRegistrationForm(UserCreationForm):
    """Form สมัครสมาชิก"""
    email = forms.EmailField(required=True, label='อีเมล')
    first_name = forms.CharField(max_length=50, label='ชื่อ')
    last_name = forms.CharField(max_length=50, label='นามสกุล')
    
    class Meta:
        model = User
        fields = ['username', 'email', 'first_name', 'last_name', 'password1', 'password2']
        labels = {
            'username': 'ชื่อผู้ใช้',
        }
    
    def clean_email(self):
        email = self.cleaned_data.get('email')
        if User.objects.filter(email=email).exists():
            raise forms.ValidationError('อีเมลนี้ถูกใช้แล้ว')
        return email
    
    def save(self, commit=True):
        user = super().save(commit=False)
        user.email = self.cleaned_data['email']
        user.first_name = self.cleaned_data['first_name']
        user.last_name = self.cleaned_data['last_name']
        if commit:
            user.save()
        return user
```

---

## 4. Login Required Decorator

```python
# views.py
from django.contrib.auth.decorators import login_required, permission_required, user_passes_test

# ต้อง login ก่อนเข้า view
@login_required
def dashboard(request):
    """หน้า dashboard - ต้อง login"""
    return render(request, 'dashboard.html')


# กำหนด login_url เอง
@login_required(login_url='/custom-login/')
def secret_view(request):
    return render(request, 'secret.html')


# ต้องมี permission
@permission_required('articles.add_article')
def create_article(request):
    """ต้องมีสิทธิ์ add_article"""
    pass


# ต้องมีหลาย permissions
@permission_required(['articles.add_article', 'articles.change_article'])
def manage_articles(request):
    pass


# Custom test function
def is_editor(user):
    """ตรวจสอบว่า user เป็น editor หรือไม่"""
    return user.groups.filter(name='Editors').exists()

@user_passes_test(is_editor, login_url='/no-permission/')
def editor_panel(request):
    """เฉพาะ Editors เท่านั้น"""
    pass
```

### Login Required Mixin สำหรับ Class-based Views

```python
from django.contrib.auth.mixins import LoginRequiredMixin, PermissionRequiredMixin
from django.views.generic import ListView, CreateView

class DashboardView(LoginRequiredMixin, ListView):
    """Dashboard ต้อง login"""
    login_url = '/login/'        # redirect ไปที่นี่ถ้าไม่ได้ login
    redirect_field_name = 'next' # ชื่อ parameter สำหรับ redirect after login
    
    model = Article
    template_name = 'dashboard.html'
    
    def get_queryset(self):
        # แสดงเฉพาะบทความของ user ที่ login
        return Article.objects.filter(author=self.request.user)


class CreateArticleView(PermissionRequiredMixin, CreateView):
    """สร้างบทความ - ต้องมีสิทธิ์"""
    permission_required = 'articles.add_article'
    model = Article
    form_class = ArticleForm
    template_name = 'articles/create.html'
```

---

## 5. Permissions และ Groups

```python
# Django สร้าง permissions อัตโนมัติสำหรับแต่ละ model:
# app_label.add_modelname      - สร้าง
# app_label.change_modelname   - แก้ไข
# app_label.delete_modelname   - ลบ
# app_label.view_modelname     - ดู

# ตัวอย่าง: articles.add_article, articles.change_article

# สร้าง Custom Permission ใน model
from django.db import models

class Article(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    
    class Meta:
        permissions = [
            ('publish_article', 'Can publish articles'),     # (codename, description)
            ('feature_article', 'Can feature articles'),
        ]
```

### จัดการ Groups

```python
from django.contrib.auth.models import Permission, Group
from django.contrib.contenttypes.models import ContentType

# สร้าง Group
editors_group, created = Group.objects.get_or_create(name='Editors')

# ดึง Permissions
content_type = ContentType.objects.get_for_model(Article)
permissions = Permission.objects.filter(content_type=content_type)

# เพิ่ม permission ให้ group
publish_perm = Permission.objects.get(codename='publish_article')
editors_group.permissions.add(publish_perm)

# เพิ่ม user เข้า group
user.groups.add(editors_group)
user.groups.remove(editors_group)

# ตรวจสอบ group
user.groups.filter(name='Editors').exists()
```

```python
# สร้าง Groups ผ่าน management command หรือ signal
# myapp/management/commands/create_groups.py
from django.core.management.base import BaseCommand
from django.contrib.auth.models import Group, Permission

class Command(BaseCommand):
    help = 'สร้าง default groups'
    
    def handle(self, *args, **options):
        # Editors group
        editors, _ = Group.objects.get_or_create(name='Editors')
        editor_permissions = [
            'articles.add_article',
            'articles.change_article',
            'articles.view_article',
            'articles.publish_article',
        ]
        for perm_str in editor_permissions:
            app_label, codename = perm_str.split('.')
            perm = Permission.objects.get(
                codename=codename,
                content_type__app_label=app_label
            )
            editors.permissions.add(perm)
        
        self.stdout.write(self.style.SUCCESS('สร้าง groups เรียบร้อยแล้ว'))
```

---

## 6. Custom User Model

**สำคัญมาก**: ควรสร้าง Custom User Model ตั้งแต่เริ่มโปรเจค เพราะการเปลี่ยนทีหลังยาก

```python
# accounts/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models

class User(AbstractUser):
    """Custom User Model เพิ่ม fields พิเศษ"""
    
    # เพิ่ม fields ที่ต้องการ
    phone = models.CharField(
        max_length=20,
        blank=True,
        verbose_name='เบอร์โทรศัพท์'
    )
    avatar = models.ImageField(
        upload_to='avatars/',
        null=True,
        blank=True,
        verbose_name='รูปโปรไฟล์'
    )
    bio = models.TextField(
        blank=True,
        verbose_name='ประวัติย่อ'
    )
    birth_date = models.DateField(
        null=True,
        blank=True,
        verbose_name='วันเกิด'
    )
    
    # เลือกประเภท
    ROLE_CHOICES = [
        ('student', 'นักเรียน'),
        ('teacher', 'ครู'),
        ('admin', 'ผู้ดูแลระบบ'),
    ]
    role = models.CharField(
        max_length=20,
        choices=ROLE_CHOICES,
        default='student',
        verbose_name='บทบาท'
    )
    
    # ยืนยันอีเมล
    email_verified = models.BooleanField(
        default=False,
        verbose_name='ยืนยันอีเมลแล้ว'
    )
    
    class Meta:
        verbose_name = 'ผู้ใช้'
        verbose_name_plural = 'ผู้ใช้'
    
    def __str__(self):
        return self.get_full_name() or self.username
    
    @property
    def is_teacher(self):
        return self.role == 'teacher'
    
    @property
    def full_name(self):
        return self.get_full_name() or self.username


# settings.py - ต้องกำหนดก่อน migrate ครั้งแรก!
AUTH_USER_MODEL = 'accounts.User'
```

### AbstractBaseUser สำหรับ User Model ที่ต่างออกไปมาก

```python
# accounts/models.py
from django.contrib.auth.models import AbstractBaseUser, BaseUserManager, PermissionsMixin
from django.db import models
from django.utils import timezone

class UserManager(BaseUserManager):
    """Custom manager สำหรับ User model ที่ใช้ email แทน username"""
    
    def create_user(self, email, password=None, **extra_fields):
        if not email:
            raise ValueError('ต้องระบุอีเมล')
        email = self.normalize_email(email)
        user = self.model(email=email, **extra_fields)
        user.set_password(password)
        user.save(using=self._db)
        return user
    
    def create_superuser(self, email, password=None, **extra_fields):
        extra_fields.setdefault('is_staff', True)
        extra_fields.setdefault('is_superuser', True)
        extra_fields.setdefault('is_active', True)
        return self.create_user(email, password, **extra_fields)


class EmailUser(AbstractBaseUser, PermissionsMixin):
    """User model ที่ใช้ email เป็น login identifier"""
    
    email = models.EmailField(unique=True, verbose_name='อีเมล')
    first_name = models.CharField(max_length=50, blank=True)
    last_name = models.CharField(max_length=50, blank=True)
    
    is_active = models.BooleanField(default=True)
    is_staff = models.BooleanField(default=False)
    date_joined = models.DateTimeField(default=timezone.now)
    
    objects = UserManager()
    
    # ใช้ email เป็น username field
    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['first_name', 'last_name']  # สำหรับ createsuperuser
    
    class Meta:
        verbose_name = 'ผู้ใช้'
        verbose_name_plural = 'ผู้ใช้'
    
    def __str__(self):
        return self.email
    
    def get_full_name(self):
        return f'{self.first_name} {self.last_name}'.strip()
```

---

## 7. User Profile Pattern

```python
# accounts/models.py
from django.db import models
from django.conf import settings
from django.db.models.signals import post_save
from django.dispatch import receiver

class UserProfile(models.Model):
    """Profile เพิ่มเติมสำหรับ User (One-to-One)"""
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='profile'
    )
    phone = models.CharField(max_length=20, blank=True)
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to='avatars/', null=True, blank=True)
    website = models.URLField(blank=True)
    
    # Social links
    facebook = models.URLField(blank=True)
    twitter = models.CharField(max_length=50, blank=True)
    line_id = models.CharField(max_length=50, blank=True)
    
    created_at = models.DateTimeField(auto_now_add=True)
    
    def __str__(self):
        return f'Profile ของ {self.user.username}'
    
    @property
    def avatar_url(self):
        if self.avatar:
            return self.avatar.url
        return '/static/images/default-avatar.png'


# สร้าง Profile อัตโนมัติเมื่อสร้าง User
@receiver(post_save, sender=settings.AUTH_USER_MODEL)
def create_user_profile(sender, instance, created, **kwargs):
    if created:
        UserProfile.objects.create(user=instance)

@receiver(post_save, sender=settings.AUTH_USER_MODEL)
def save_user_profile(sender, instance, **kwargs):
    instance.profile.save()
```

```python
# views.py
from django.contrib.auth.decorators import login_required
from django.shortcuts import render, redirect
from django.contrib import messages
from .forms import UserProfileForm, UserUpdateForm

@login_required
def profile_view(request):
    """หน้าโปรไฟล์"""
    return render(request, 'accounts/profile.html', {
        'user': request.user,
        'profile': request.user.profile
    })

@login_required
def profile_edit(request):
    """แก้ไขโปรไฟล์"""
    if request.method == 'POST':
        user_form = UserUpdateForm(request.POST, instance=request.user)
        profile_form = UserProfileForm(
            request.POST,
            request.FILES,
            instance=request.user.profile
        )
        
        if user_form.is_valid() and profile_form.is_valid():
            user_form.save()
            profile_form.save()
            messages.success(request, 'อัปเดตโปรไฟล์เรียบร้อยแล้ว')
            return redirect('profile')
    else:
        user_form = UserUpdateForm(instance=request.user)
        profile_form = UserProfileForm(instance=request.user.profile)
    
    return render(request, 'accounts/profile_edit.html', {
        'user_form': user_form,
        'profile_form': profile_form
    })
```

---

## 8. Template Tags สำหรับ Auth

```html
<!-- ตรวจสอบว่า login แล้วหรือยัง -->
{% if user.is_authenticated %}
<p>สวัสดี {{ user.get_full_name }}</p>
<a href="{% url 'logout' %}">ออกจากระบบ</a>
{% else %}
<a href="{% url 'login' %}">เข้าสู่ระบบ</a>
{% endif %}

<!-- ตรวจสอบ permission ใน template -->
{% if perms.articles.add_article %}
<a href="{% url 'article_create' %}">เพิ่มบทความ</a>
{% endif %}

{% if perms.articles %}
<!-- user มี permission ใดๆ ใน articles app -->
<a href="{% url 'article_list' %}">จัดการบทความ</a>
{% endif %}

<!-- ตรวจสอบ staff/superuser -->
{% if user.is_staff %}
<a href="/admin/">Admin Panel</a>
{% endif %}
```

---

## 9. สรุป Part 063

✅ **Django Authentication** มี User model, sessions, permissions built-in
✅ **Login/Logout** ใช้ built-in views หรือสร้าง custom views เอง
✅ **@login_required** ป้องกัน view ที่ต้องการ authentication
✅ **Permissions** ใช้ `@permission_required` และ `user.has_perm()`
✅ **Groups** รวม permissions หลายอย่างและกำหนดให้ user ทั้งกลุ่ม
✅ **AbstractUser** เพิ่ม fields พิเศษใน User model
✅ **AbstractBaseUser** สร้าง User model ใหม่ทั้งหมด (เช่น ใช้ email แทน username)
✅ ควรกำหนด `AUTH_USER_MODEL` ตั้งแต่เริ่มโปรเจค

## ➡️ ถัดไป: Part 064 - Django REST Framework (DRF) Setup

*Part 063/100+ | Python Course - Beginner to World-Class*
