# Part 59: Django Authentication

## เป้าหมายของบทเรียน

- ใช้ Django built-in authentication
- จัดการ User model (login, logout, register)
- Password management
- Permission system
- ใช้ @login_required decorator
- สร้าง Custom User Model

---

## 1. Django Authentication System

Django มีระบบ authentication ในตัวที่ประกอบด้วย:
- User model
- Permissions และ Groups
- Password hashing
- Login/Logout views
- Middleware สำหรับ request.user

```python
# settings.py - ตรวจสอบว่ามี auth apps
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',        # Authentication framework
    'django.contrib.contenttypes',
    # ...
]

MIDDLEWARE = [
    # ...
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',  # ใส่ user ใน request
    # ...
]
```

---

## 2. User Model

```python
# Django built-in User model fields:
# - username (unique, max 150 chars)
# - first_name, last_name
# - email
# - password (hashed)
# - groups (ManyToMany)
# - user_permissions (ManyToMany)
# - is_staff (Boolean)
# - is_active (Boolean)
# - is_superuser (Boolean)
# - last_login (DateTimeField)
# - date_joined (DateTimeField)

from django.contrib.auth.models import User

# สร้าง regular user
user = User.objects.create_user(
    username='somchai',
    email='somchai@example.com',
    password='password123',
    first_name='สมชาย',
    last_name='ใจดี',
)

# สร้าง superuser
superuser = User.objects.create_superuser(
    username='admin',
    email='admin@example.com',
    password='adminpassword',
)

# ดึง user
user = User.objects.get(username='somchai')

# ตรวจสอบ password
user.check_password('password123')  # True
user.check_password('wrong')        # False

# เปลี่ยน password
user.set_password('newpassword123')
user.save()

# User methods
user.get_full_name()          # 'สมชาย ใจดี'
user.get_short_name()         # 'สมชาย'
user.get_username()           # 'somchai'

# Permission checks
user.is_active                # True/False
user.is_staff                 # True/False
user.is_superuser             # True/False
user.is_authenticated         # True (เสมอสำหรับ User object)
```

---

## 3. Login และ Logout

### ใช้ Django built-in views

```python
# mysite/urls.py
from django.contrib.auth import views as auth_views
from django.urls import path, include

urlpatterns = [
    # Django built-in auth URLs
    path('accounts/', include('django.contrib.auth.urls')),
    # สร้าง URLs:
    # /accounts/login/          -> login
    # /accounts/logout/         -> logout
    # /accounts/password_change/
    # /accounts/password_change/done/
    # /accounts/password_reset/
    # /accounts/password_reset/done/
    # /accounts/reset/<uidb64>/<token>/
    # /accounts/reset/done/
    
    # หรือกำหนด view เองได้
    path('login/', auth_views.LoginView.as_view(
        template_name='accounts/login.html'
    ), name='login'),
    path('logout/', auth_views.LogoutView.as_view(), name='logout'),
]
```

```python
# settings.py
# URL redirect หลัง login
LOGIN_URL = '/accounts/login/'
LOGIN_REDIRECT_URL = '/blog/'
LOGOUT_REDIRECT_URL = '/'
```

```html
<!-- templates/accounts/login.html -->
{% extends 'base.html' %}

{% block title %}เข้าสู่ระบบ{% endblock %}

{% block content %}
<div class="row justify-content-center mt-5">
    <div class="col-md-5">
        <div class="card shadow">
            <div class="card-header text-center">
                <h4>เข้าสู่ระบบ</h4>
            </div>
            <div class="card-body p-4">
                <form method="post">
                    {% csrf_token %}
                    
                    {% if form.errors %}
                    <div class="alert alert-danger">
                        ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง
                    </div>
                    {% endif %}
                    
                    <div class="mb-3">
                        <label for="id_username" class="form-label">ชื่อผู้ใช้</label>
                        <input type="text" 
                               name="username" 
                               id="id_username"
                               class="form-control" 
                               autofocus required>
                    </div>
                    
                    <div class="mb-3">
                        <label for="id_password" class="form-label">รหัสผ่าน</label>
                        <input type="password" 
                               name="password" 
                               id="id_password"
                               class="form-control" 
                               required>
                    </div>
                    
                    <!-- hidden field สำหรับ redirect URL -->
                    <input type="hidden" name="next" value="{{ next }}">
                    
                    <div class="d-grid">
                        <button type="submit" class="btn btn-primary">
                            เข้าสู่ระบบ
                        </button>
                    </div>
                </form>
                
                <hr>
                <div class="text-center">
                    <a href="{% url 'password_reset' %}">ลืมรหัสผ่าน?</a>
                    &nbsp;|&nbsp;
                    <a href="{% url 'register' %}">สมัครสมาชิก</a>
                </div>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

### Manual Login/Logout ใน View

```python
# accounts/views.py
from django.contrib.auth import authenticate, login, logout
from django.shortcuts import render, redirect
from django.contrib import messages


def custom_login(request):
    """
    Custom login view
    """
    if request.user.is_authenticated:
        return redirect('blog:post_list')
    
    if request.method == 'POST':
        username = request.POST.get('username')
        password = request.POST.get('password')
        
        # authenticate() ตรวจสอบ credentials และ return User หรือ None
        user = authenticate(request, username=username, password=password)
        
        if user is not None:
            if user.is_active:
                # login() สร้าง session
                login(request, user)
                
                # redirect ไปยัง 'next' URL ถ้ามี
                next_url = request.POST.get('next', request.GET.get('next', ''))
                if next_url and next_url.startswith('/'):
                    return redirect(next_url)
                
                messages.success(request, f'ยินดีต้อนรับกลับมา, {user.get_full_name() or user.username}!')
                return redirect('blog:post_list')
            else:
                messages.error(request, 'บัญชีของคุณถูกระงับ')
        else:
            messages.error(request, 'ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง')
    
    return render(request, 'accounts/login.html')


def custom_logout(request):
    """Logout view"""
    if request.method == 'POST':
        logout(request)  # ลบ session
        messages.success(request, 'ออกจากระบบสำเร็จ')
    return redirect('pages:home')
```

---

## 4. Registration

```python
# accounts/views.py
from django.contrib.auth.models import User
from django.contrib.auth import login
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
            messages.success(
                request,
                f'สมัครสมาชิกสำเร็จ! ยินดีต้อนรับ {user.get_full_name() or user.username}'
            )
            return redirect('blog:post_list')
    else:
        form = RegisterForm()
    
    return render(request, 'accounts/register.html', {
        'form': form,
        'page_title': 'สมัครสมาชิก',
    })
```

---

## 5. Password Management

### Built-in Password Reset

```python
# settings.py - Email configuration
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = 'smtp.gmail.com'
EMAIL_PORT = 587
EMAIL_USE_TLS = True
EMAIL_HOST_USER = 'your-email@gmail.com'
EMAIL_HOST_PASSWORD = 'your-app-password'
DEFAULT_FROM_EMAIL = 'noreply@myblog.com'

# ใน development ใช้ console backend แทน
# EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
```

```python
# urls.py
from django.contrib.auth import views as auth_views

urlpatterns = [
    path('accounts/', include('django.contrib.auth.urls')),
    # หรือกำหนด template เอง:
    path('password/reset/', 
         auth_views.PasswordResetView.as_view(
             template_name='accounts/password_reset.html',
             email_template_name='accounts/password_reset_email.html',
             subject_template_name='accounts/password_reset_subject.txt',
         ), 
         name='password_reset'),
    path('password/reset/done/',
         auth_views.PasswordResetDoneView.as_view(
             template_name='accounts/password_reset_done.html'
         ),
         name='password_reset_done'),
    path('password/reset/<uidb64>/<token>/',
         auth_views.PasswordResetConfirmView.as_view(
             template_name='accounts/password_reset_confirm.html'
         ),
         name='password_reset_confirm'),
    path('password/reset/complete/',
         auth_views.PasswordResetCompleteView.as_view(
             template_name='accounts/password_reset_complete.html'
         ),
         name='password_reset_complete'),
]
```

### Password Change

```python
from django.contrib.auth.decorators import login_required
from django.contrib.auth import update_session_auth_hash
from django.contrib.auth.forms import PasswordChangeForm


@login_required
def change_password(request):
    """เปลี่ยนรหัสผ่าน"""
    if request.method == 'POST':
        form = PasswordChangeForm(request.user, request.POST)
        if form.is_valid():
            user = form.save()
            # อัปเดต session เพื่อไม่ให้ logout หลังเปลี่ยน password
            update_session_auth_hash(request, user)
            messages.success(request, 'เปลี่ยนรหัสผ่านสำเร็จ!')
            return redirect('accounts:profile')
        else:
            messages.error(request, 'กรุณาตรวจสอบข้อมูล')
    else:
        form = PasswordChangeForm(request.user)
    
    return render(request, 'accounts/change_password.html', {'form': form})
```

---

## 6. @login_required Decorator

```python
from django.contrib.auth.decorators import login_required
from django.contrib.auth.mixins import LoginRequiredMixin


# สำหรับ FBV
@login_required
def create_post(request):
    """ต้อง login ก่อนเข้าถึง"""
    pass


# กำหนด login URL
@login_required(login_url='/accounts/login/')
def profile(request):
    pass


# กำหนด redirect URL (URL ที่จะ redirect หลัง login สำเร็จ)
@login_required(redirect_field_name='redirect_to')
def dashboard(request):
    pass


# สำหรับ CBV
class CreatePostView(LoginRequiredMixin, CreateView):
    login_url = '/accounts/login/'
    redirect_field_name = 'next'
    # ...
```

---

## 7. Permission System

### Built-in Permissions

Django สร้าง permissions อัตโนมัติสำหรับทุก model:
- `app_label.add_modelname`
- `app_label.change_modelname`
- `app_label.delete_modelname`
- `app_label.view_modelname`

```python
# ตรวจสอบ permission
user.has_perm('blog.add_post')      # True/False
user.has_perm('blog.change_post')
user.has_perm('blog.delete_post')
user.has_perm('blog.view_post')

# ตรวจสอบหลาย permissions
user.has_perms(['blog.add_post', 'blog.change_post'])

# superuser มีทุก permissions
superuser.has_perm('anything')  # True

# Inactive user ไม่มี permissions
inactive_user.has_perm('blog.add_post')  # False
```

### Assign Permissions

```python
from django.contrib.auth.models import Permission
from django.contrib.contenttypes.models import ContentType


# ดึง permission
content_type = ContentType.objects.get_for_model(Post)
permission = Permission.objects.get(
    codename='add_post',
    content_type=content_type,
)

# กำหนด permission ให้ user
user.user_permissions.add(permission)
user.user_permissions.remove(permission)
user.user_permissions.set([permission1, permission2])
user.user_permissions.clear()

# ตรวจสอบ (ต้อง refetch จาก database หรือ clear cache)
from django.contrib.auth.models import User
user = User.objects.get(pk=user.pk)  # refetch
user.has_perm('blog.add_post')
```

### Groups

```python
from django.contrib.auth.models import Group


# สร้าง group
editors_group, created = Group.objects.get_or_create(name='Editors')

# เพิ่ม permissions ให้ group
editors_group.permissions.add(
    Permission.objects.get(codename='add_post'),
    Permission.objects.get(codename='change_post'),
    Permission.objects.get(codename='view_post'),
)

# เพิ่ม user เข้า group
user.groups.add(editors_group)
user.groups.remove(editors_group)

# ตรวจสอบ group
user.groups.filter(name='Editors').exists()
```

### Permission Decorators

```python
from django.contrib.auth.decorators import permission_required
from django.contrib.auth.mixins import PermissionRequiredMixin


# สำหรับ FBV
@permission_required('blog.add_post')
def create_post(request):
    pass


# ถ้าไม่มี permission raise PermissionDenied (403)
@permission_required('blog.add_post', raise_exception=True)
def create_post(request):
    pass


# หลาย permissions
@permission_required(['blog.add_post', 'blog.change_post'])
def manage_post(request):
    pass


# สำหรับ CBV
class CreatePostView(PermissionRequiredMixin, CreateView):
    permission_required = 'blog.add_post'
    # หรือหลาย permissions:
    # permission_required = ['blog.add_post', 'blog.change_post']


# ตรวจสอบใน template
# {% if user.has_perm('blog.add_post') %}
# <a href="{% url 'blog:post_create' %}">สร้างบทความ</a>
# {% endif %}
```

### Custom Permissions

```python
# models.py
class Post(models.Model):
    # ...
    
    class Meta:
        # กำหนด custom permissions
        permissions = [
            ('publish_post', 'สามารถเผยแพร่บทความได้'),
            ('feature_post', 'สามารถตั้งเป็น featured ได้'),
            ('export_posts', 'สามารถ export บทความได้'),
        ]
```

```bash
# หลังเพิ่ม permissions ต้อง migrate
python manage.py makemigrations
python manage.py migrate
```

```python
# ใช้ custom permission
@permission_required('blog.publish_post')
def publish_post(request, slug):
    post = get_object_or_404(Post, slug=slug)
    post.publish()
    return redirect('blog:post_detail', slug=post.slug)
```

---

## 8. Custom User Model

แนะนำให้สร้าง Custom User Model ตั้งแต่เริ่มต้น project

### AbstractUser (ง่ายสุด - extend User เดิม)

```python
# accounts/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models


class User(AbstractUser):
    """
    Custom User Model
    เพิ่ม fields ใหม่ที่ต้องการ
    """
    # fields เพิ่มเติม
    bio = models.TextField(blank=True, verbose_name='ประวัติโดยย่อ')
    avatar = models.ImageField(
        upload_to='avatars/',
        null=True,
        blank=True,
        verbose_name='รูปโปรไฟล์'
    )
    website = models.URLField(blank=True, verbose_name='เว็บไซต์')
    location = models.CharField(max_length=100, blank=True, verbose_name='ที่อยู่')
    phone = models.CharField(max_length=20, blank=True, verbose_name='เบอร์โทร')
    
    # เปลี่ยน email ให้ unique
    email = models.EmailField(unique=True)
    
    class Meta:
        verbose_name = 'ผู้ใช้'
        verbose_name_plural = 'ผู้ใช้ทั้งหมด'
    
    def __str__(self):
        return self.get_full_name() or self.username
    
    @property
    def full_name(self):
        return self.get_full_name()
    
    def get_avatar_url(self):
        """คืน URL ของ avatar หรือ default"""
        if self.avatar:
            return self.avatar.url
        # gravatar หรือ default
        return f'https://ui-avatars.com/api/?name={self.username}&background=random'
```

```python
# settings.py
# บอก Django ให้ใช้ Custom User Model
AUTH_USER_MODEL = 'accounts.User'

# ต้องกำหนดก่อน migrate ครั้งแรก!
```

```python
# accounts/admin.py
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin
from .models import User


@admin.register(User)
class CustomUserAdmin(UserAdmin):
    """Admin สำหรับ Custom User"""
    list_display = ['username', 'email', 'full_name', 'is_staff', 'date_joined']
    list_filter = ['is_staff', 'is_superuser', 'is_active', 'date_joined']
    search_fields = ['username', 'email', 'first_name', 'last_name']
    ordering = ['-date_joined']
    
    # เพิ่ม fields ใน form
    fieldsets = UserAdmin.fieldsets + (
        ('ข้อมูลเพิ่มเติม', {
            'fields': ['bio', 'avatar', 'website', 'location', 'phone']
        }),
    )
    
    # fields สำหรับสร้าง user ใหม่
    add_fieldsets = UserAdmin.add_fieldsets + (
        ('ข้อมูลส่วนตัว', {
            'fields': ['email', 'first_name', 'last_name']
        }),
    )
```

### AbstractBaseUser (ควบคุมสูงสุด)

```python
# accounts/models.py
from django.contrib.auth.models import AbstractBaseUser, BaseUserManager, PermissionsMixin
from django.db import models


class UserManager(BaseUserManager):
    """Custom manager สำหรับ User"""
    
    def create_user(self, email, password=None, **extra_fields):
        """สร้าง regular user"""
        if not email:
            raise ValueError('ต้องมี email')
        email = self.normalize_email(email)
        extra_fields.setdefault('is_active', True)
        user = self.model(email=email, **extra_fields)
        user.set_password(password)
        user.save(using=self._db)
        return user
    
    def create_superuser(self, email, password=None, **extra_fields):
        """สร้าง superuser"""
        extra_fields.setdefault('is_staff', True)
        extra_fields.setdefault('is_superuser', True)
        extra_fields.setdefault('is_active', True)
        
        if not extra_fields.get('is_staff'):
            raise ValueError('Superuser ต้องมี is_staff=True')
        if not extra_fields.get('is_superuser'):
            raise ValueError('Superuser ต้องมี is_superuser=True')
        
        return self.create_user(email, password, **extra_fields)


class User(AbstractBaseUser, PermissionsMixin):
    """
    Custom User Model ที่ใช้ email แทน username
    """
    email = models.EmailField(unique=True, verbose_name='อีเมล')
    first_name = models.CharField(max_length=150, verbose_name='ชื่อ')
    last_name = models.CharField(max_length=150, verbose_name='นามสกุล')
    bio = models.TextField(blank=True, verbose_name='ประวัติ')
    avatar = models.ImageField(upload_to='avatars/', null=True, blank=True)
    
    is_active = models.BooleanField(default=True)
    is_staff = models.BooleanField(default=False)
    date_joined = models.DateTimeField(auto_now_add=True)
    
    objects = UserManager()
    
    # กำหนด field ที่ใช้เป็น username
    USERNAME_FIELD = 'email'
    
    # fields ที่ต้องกรอกเมื่อ createsuperuser
    REQUIRED_FIELDS = ['first_name', 'last_name']
    
    class Meta:
        verbose_name = 'ผู้ใช้'
        verbose_name_plural = 'ผู้ใช้ทั้งหมด'
    
    def __str__(self):
        return self.email
    
    def get_full_name(self):
        return f'{self.first_name} {self.last_name}'.strip()
    
    def get_short_name(self):
        return self.first_name
```

---

## 9. User Profile (OneToOne Extension)

แทนที่จะ extend User model อาจใช้ Profile model แทน

```python
# accounts/models.py
from django.conf import settings
from django.db import models
from django.db.models.signals import post_save
from django.dispatch import receiver


class Profile(models.Model):
    """Profile เสริมสำหรับ User"""
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='profile'
    )
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to='avatars/', null=True, blank=True)
    website = models.URLField(blank=True)
    location = models.CharField(max_length=100, blank=True)
    birth_date = models.DateField(null=True, blank=True)
    
    # Social links
    twitter = models.CharField(max_length=100, blank=True)
    instagram = models.CharField(max_length=100, blank=True)
    facebook = models.CharField(max_length=200, blank=True)
    
    class Meta:
        verbose_name = 'โปรไฟล์'
    
    def __str__(self):
        return f'โปรไฟล์ของ {self.user.username}'


# สร้าง Profile อัตโนมัติเมื่อสร้าง User
@receiver(post_save, sender=settings.AUTH_USER_MODEL)
def create_user_profile(sender, instance, created, **kwargs):
    if created:
        Profile.objects.create(user=instance)


@receiver(post_save, sender=settings.AUTH_USER_MODEL)
def save_user_profile(sender, instance, **kwargs):
    if hasattr(instance, 'profile'):
        instance.profile.save()
```

```python
# views.py
@login_required
def profile_view(request):
    """แสดงและแก้ไข profile"""
    user = request.user
    profile = user.profile
    
    if request.method == 'POST':
        user_form = UserUpdateForm(request.POST, instance=user)
        profile_form = ProfileUpdateForm(
            request.POST, 
            request.FILES, 
            instance=profile
        )
        
        if user_form.is_valid() and profile_form.is_valid():
            user_form.save()
            profile_form.save()
            messages.success(request, 'อัปเดต profile สำเร็จ!')
            return redirect('accounts:profile')
    else:
        user_form = UserUpdateForm(instance=user)
        profile_form = ProfileUpdateForm(instance=profile)
    
    return render(request, 'accounts/profile.html', {
        'user_form': user_form,
        'profile_form': profile_form,
    })
```

---

## 10. ตัวอย่างเต็ม: Accounts App

```python
# accounts/urls.py
from django.urls import path
from django.contrib.auth import views as auth_views
from . import views

app_name = 'accounts'

urlpatterns = [
    path('register/', views.register, name='register'),
    path('login/', views.custom_login, name='login'),
    path('logout/', views.custom_logout, name='logout'),
    path('profile/', views.profile_view, name='profile'),
    path('profile/edit/', views.profile_edit, name='profile_edit'),
    path('change-password/', views.change_password, name='change_password'),
    
    # Password reset
    path('password/reset/', 
         auth_views.PasswordResetView.as_view(
             template_name='accounts/password_reset.html'
         ), 
         name='password_reset'),
    path('password/reset/done/',
         auth_views.PasswordResetDoneView.as_view(
             template_name='accounts/password_reset_done.html'
         ),
         name='password_reset_done'),
    path('password/reset/confirm/<uidb64>/<token>/',
         auth_views.PasswordResetConfirmView.as_view(
             template_name='accounts/password_reset_confirm.html'
         ),
         name='password_reset_confirm'),
    path('password/reset/complete/',
         auth_views.PasswordResetCompleteView.as_view(
             template_name='accounts/password_reset_complete.html'
         ),
         name='password_reset_complete'),
]
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Email Verification
สร้างระบบ email verification:
- หลังสมัครสมาชิก ส่ง verification email
- User ต้อง click link ใน email เพื่อ activate account
- ใช้ `is_active=False` จนกว่าจะ verify

### แบบฝึกหัดที่ 2: Role-Based Access Control
สร้างระบบ RBAC:
- สร้าง Groups: Admin, Editor, Author, Reader
- กำหนด permissions ให้แต่ละ group
- Middleware ที่ตรวจสอบ group ก่อนเข้า views บางส่วน

### แบบฝึกหัดที่ 3: Social Authentication
ติดตั้ง `django-allauth` เพื่อ:
- Login ด้วย Google
- Login ด้วย GitHub
- Connect/Disconnect social accounts

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- Django built-in authentication
- User model และ methods
- Login, Logout, Registration
- Password management
- Permission system (built-in และ custom)
- @login_required decorator
- Custom User Model (AbstractUser, AbstractBaseUser)
- User Profile pattern

---

## บทถัดไป

➡️ **[Part 60: Django REST Framework](part-060.md)** - เรียนรู้การสร้าง REST API ด้วย Django REST Framework
