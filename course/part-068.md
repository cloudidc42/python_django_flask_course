# Part 068: DRF Permissions

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Permission system ใน DRF
- ใช้ built-in permissions เช่น IsAuthenticated, IsAdminUser, AllowAny
- สร้าง Custom Permissions
- ทำ Object-level permissions
- รวม permissions หลายอัน

---

## 1. Permissions คืออะไร?

Permission ตรวจสอบว่า user มีสิทธิ์ทำ request นั้นหรือไม่

```
Request -> Authentication -> Permission -> View
                |                |
           (ใครคุณ)         (ทำได้ไหม)
```

DRF ตรวจสอบ permissions ใน 2 ระดับ:
1. **View-level**: ตรวจก่อนเข้า view (has_permission)
2. **Object-level**: ตรวจก่อนเข้าถึง object เฉพาะชิ้น (has_object_permission)

---

## 2. Built-in Permissions

```python
from rest_framework.permissions import (
    AllowAny,              # ทุกคนเข้าได้
    IsAuthenticated,       # ต้อง login
    IsAdminUser,           # ต้องเป็น staff
    IsAuthenticatedOrReadOnly,  # login ถึงแก้ไขได้ ไม่ login อ่านได้
    DjangoModelPermissions,     # ใช้ Django model permissions
    DjangoObjectPermissions,    # Object-level Django permissions
)
```

### AllowAny

```python
from rest_framework.permissions import AllowAny
from rest_framework.views import APIView

class PublicView(APIView):
    permission_classes = [AllowAny]  # ทุกคนเข้าได้ ไม่ต้อง login
    
    def get(self, request):
        return Response({'message': 'สาธารณะ'})
```

### IsAuthenticated

```python
from rest_framework.permissions import IsAuthenticated

class PrivateView(APIView):
    permission_classes = [IsAuthenticated]  # ต้อง login เท่านั้น
    
    def get(self, request):
        return Response({
            'message': f'สวัสดี {request.user.username}'
        })
```

### IsAdminUser

```python
from rest_framework.permissions import IsAdminUser

class AdminView(APIView):
    permission_classes = [IsAdminUser]  # ต้องเป็น is_staff=True
    
    def get(self, request):
        return Response({'data': 'ข้อมูลสำหรับ admin'})
```

### IsAuthenticatedOrReadOnly

```python
from rest_framework.permissions import IsAuthenticatedOrReadOnly

class ArticleViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticatedOrReadOnly]
    # GET (list, retrieve) - ทุกคนเข้าได้
    # POST, PUT, PATCH, DELETE - ต้อง login
```

---

## 3. Custom Permissions

```python
# permissions.py
from rest_framework import permissions

class IsOwnerOrReadOnly(permissions.BasePermission):
    """
    Object-level permission
    ให้แก้ไขได้เฉพาะ owner
    """
    
    def has_permission(self, request, view):
        """View-level: ตรวจสอบก่อนเข้า view"""
        # อ่านได้ทุกคน
        if request.method in permissions.SAFE_METHODS:
            return True
        # เขียนต้อง login
        return request.user and request.user.is_authenticated
    
    def has_object_permission(self, request, view, obj):
        """Object-level: ตรวจสอบก่อนเข้าถึง object"""
        # อ่านได้ทุกคน
        if request.method in permissions.SAFE_METHODS:
            return True
        
        # เขียนต้องเป็น owner หรือ admin
        if request.user.is_staff:
            return True
        
        # ตรวจสอบว่า object มี field 'author' หรือ 'owner'
        if hasattr(obj, 'author'):
            return obj.author == request.user
        if hasattr(obj, 'owner'):
            return obj.owner == request.user
        if hasattr(obj, 'user'):
            return obj.user == request.user
        
        return False


class IsEmailVerified(permissions.BasePermission):
    """ต้องยืนยันอีเมลแล้ว"""
    message = 'กรุณายืนยันอีเมลของคุณก่อน'
    
    def has_permission(self, request, view):
        if not request.user.is_authenticated:
            return False
        
        # ตรวจสอบจาก profile หรือ field ใน user model
        if hasattr(request.user, 'email_verified'):
            return request.user.email_verified
        
        if hasattr(request.user, 'profile'):
            return request.user.profile.email_verified
        
        return True


class IsAdminOrOwner(permissions.BasePermission):
    """Admin มีสิทธิ์ทุกอย่าง Owner มีสิทธิ์เฉพาะของตัวเอง"""
    
    def has_permission(self, request, view):
        return request.user and request.user.is_authenticated
    
    def has_object_permission(self, request, view, obj):
        if request.user.is_staff:
            return True
        
        owner_field = getattr(view, 'owner_field', 'user')
        owner = getattr(obj, owner_field, None)
        return owner == request.user


class RoleBasedPermission(permissions.BasePermission):
    """Permission ตาม role"""
    
    # กำหนด roles ที่อนุญาตสำหรับแต่ละ HTTP method
    SAFE_METHODS_ROLES = ['student', 'teacher', 'admin']
    WRITE_METHODS_ROLES = ['teacher', 'admin']
    DELETE_METHODS_ROLES = ['admin']
    
    def has_permission(self, request, view):
        if not request.user.is_authenticated:
            return False
        
        user_role = getattr(request.user, 'role', None)
        
        if request.method in permissions.SAFE_METHODS:
            return user_role in self.SAFE_METHODS_ROLES
        
        if request.method == 'DELETE':
            return user_role in self.DELETE_METHODS_ROLES
        
        return user_role in self.WRITE_METHODS_ROLES


class HasAPIKey(permissions.BasePermission):
    """ต้องมี API Key"""
    message = 'ต้องระบุ API Key'
    
    def has_permission(self, request, view):
        api_key = request.META.get('HTTP_X_API_KEY')
        if not api_key:
            return False
        
        from .models import APIKey
        return APIKey.objects.filter(
            key=api_key,
            is_active=True
        ).exists()


class IsArticleAuthorOrAdmin(permissions.BasePermission):
    """เฉพาะ author ของบทความหรือ admin"""
    
    def has_object_permission(self, request, view, obj):
        # อ่านได้ทุกคน
        if request.method in permissions.SAFE_METHODS:
            return True
        
        # admin ทำได้ทุกอย่าง
        if request.user.is_staff or request.user.is_superuser:
            return True
        
        # author ทำได้เฉพาะบทความตัวเอง
        return obj.author == request.user


class SubscriptionPermission(permissions.BasePermission):
    """ต้อง subscribe ก่อนจึงจะเข้าถึง premium content"""
    message = 'กรุณา subscribe เพื่อเข้าถึง premium content'
    
    def has_permission(self, request, view):
        if not request.user.is_authenticated:
            return False
        
        # ตรวจสอบ subscription
        try:
            subscription = request.user.subscription
            return subscription.is_active and not subscription.is_expired
        except Exception:
            return False
```

---

## 4. กำหนด Permissions ต่อ View

```python
from rest_framework import viewsets
from .permissions import IsOwnerOrReadOnly, IsAdminOrOwner

class ArticleViewSet(viewsets.ModelViewSet):
    
    def get_permissions(self):
        """กำหนด permissions ตาม action"""
        if self.action in ['list', 'retrieve']:
            # อ่านได้ทุกคน
            permission_classes = [AllowAny]
        elif self.action == 'create':
            # สร้างต้อง login
            permission_classes = [IsAuthenticated]
        elif self.action in ['update', 'partial_update', 'destroy']:
            # แก้ไข/ลบต้องเป็น owner หรือ admin
            permission_classes = [IsAuthenticated, IsOwnerOrReadOnly]
        else:
            permission_classes = [IsAuthenticated]
        
        return [permission() for permission in permission_classes]
```

---

## 5. Object-level Permissions

Object-level permissions จะถูก call เมื่อเรียก `self.get_object()` ใน ViewSet

```python
class CommentViewSet(viewsets.ModelViewSet):
    queryset = Comment.objects.all()
    serializer_class = CommentSerializer
    permission_classes = [IsAuthenticated, IsOwnerOrReadOnly]
    
    def get_object(self):
        """get_object() จะเรียก check_object_permissions อัตโนมัติ"""
        obj = super().get_object()
        # check_object_permissions ถูกเรียกโดยอัตโนมัติใน get_object()
        return obj
    
    # สำหรับ APIView ต้อง call เอง
    def retrieve(self, request, pk=None):
        obj = get_object_or_404(Comment, pk=pk)
        # ต้อง call check_object_permissions เอง
        self.check_object_permissions(request, obj)
        serializer = CommentSerializer(obj)
        return Response(serializer.data)
```

---

## 6. รวม Permissions หลายอัน

```python
from rest_framework.permissions import IsAuthenticated, IsAdminUser, AllowAny

# AND - ทุก permission ต้องผ่าน
class MyView(APIView):
    permission_classes = [IsAuthenticated, IsEmailVerified]
    # ต้องผ่านทั้งคู่

# OR - ผ่านอันใดอันหนึ่งก็ได้ (DRF 3.9+)
from rest_framework.permissions import OR

class MyView(APIView):
    permission_classes = [IsAdminUser | IsOwnerOrReadOnly]

# NOT
class MyView(APIView):
    permission_classes = [~IsAdminUser]  # ไม่ใช่ admin

# รวม
class MyView(APIView):
    permission_classes = [(IsAdminUser | IsOwnerOrReadOnly) & IsEmailVerified]
```

---

## 7. Default Permissions ใน Settings

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
        # default ทุก view ต้อง login
    ],
}
```

---

## 8. Permission Error Messages

```python
class CustomPermission(permissions.BasePermission):
    # กำหนด default message
    message = 'ไม่มีสิทธิ์เข้าถึง'
    
    def has_permission(self, request, view):
        if not request.user.is_authenticated:
            # เปลี่ยน message ตาม case
            self.message = 'กรุณาเข้าสู่ระบบก่อน'
            return False
        
        if not request.user.is_active:
            self.message = 'บัญชีถูกระงับ'
            return False
        
        return True
```

---

## 9. DjangoModelPermissions

```python
from rest_framework.permissions import DjangoModelPermissions

class ArticleViewSet(viewsets.ModelViewSet):
    """ใช้ Django model permissions"""
    permission_classes = [DjangoModelPermissions]
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    
    # DjangoModelPermissions map HTTP methods กับ Django permissions:
    # GET    -> view permission  (articles.view_article)
    # POST   -> add permission   (articles.add_article)
    # PUT    -> change permission (articles.change_article)
    # PATCH  -> change permission (articles.change_article)
    # DELETE -> delete permission (articles.delete_article)
```

---

## 10. ตัวอย่างการใช้งาน Permissions จริง

### Blog API Permissions

```python
# permissions.py
from rest_framework import permissions

class ArticlePermission(permissions.BasePermission):
    """
    Permission สำหรับ Blog API:
    - GET: ทุกคนอ่านได้ (published เท่านั้น)
    - POST: ต้อง login และมี role teacher/admin
    - PUT/PATCH: เฉพาะ author หรือ admin
    - DELETE: เฉพาะ admin
    """
    
    def has_permission(self, request, view):
        # อ่านได้ทุกคน
        if request.method in permissions.SAFE_METHODS:
            return True
        
        # ต้อง login สำหรับ write operations
        if not request.user.is_authenticated:
            return False
        
        # DELETE เฉพาะ admin
        if request.method == 'DELETE':
            return request.user.is_staff
        
        # POST ต้องมี role ที่เหมาะสม
        if request.method == 'POST':
            allowed_roles = ['teacher', 'admin']
            return getattr(request.user, 'role', '') in allowed_roles or request.user.is_staff
        
        return True
    
    def has_object_permission(self, request, view, obj):
        # อ่านได้ทุกคน
        if request.method in permissions.SAFE_METHODS:
            return True
        
        # DELETE เฉพาะ admin
        if request.method == 'DELETE':
            return request.user.is_staff
        
        # PUT/PATCH เฉพาะ author หรือ admin
        return obj.author == request.user or request.user.is_staff


class CommentPermission(permissions.BasePermission):
    """Permission สำหรับ Comments"""
    
    def has_permission(self, request, view):
        if request.method in permissions.SAFE_METHODS:
            return True
        return request.user and request.user.is_authenticated
    
    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:
            return True
        # แก้ไข/ลบได้เฉพาะเจ้าของ comment
        return obj.author == request.user or request.user.is_staff
```

```python
# ใช้ใน ViewSet
class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [ArticlePermission]
    
    def get_queryset(self):
        queryset = Article.objects.all()
        # ถ้าไม่ใช่ staff แสดงเฉพาะ published
        if not self.request.user.is_staff:
            queryset = queryset.filter(status='published')
        return queryset
```

---

## 11. Permission Testing

```python
# tests/test_permissions.py
from rest_framework.test import APITestCase
from rest_framework import status
from django.contrib.auth import get_user_model

User = get_user_model()

class ArticlePermissionTest(APITestCase):
    
    def setUp(self):
        self.regular_user = User.objects.create_user(
            username='regular', password='pass123'
        )
        self.author = User.objects.create_user(
            username='author', password='pass123'
        )
        self.admin = User.objects.create_user(
            username='admin', password='pass123', is_staff=True
        )
        self.article = Article.objects.create(
            title='Test', slug='test',
            author=self.author, content='Content',
            status='published'
        )
    
    def test_anonymous_can_read(self):
        response = self.client.get(f'/api/articles/{self.article.pk}/')
        self.assertEqual(response.status_code, status.HTTP_200_OK)
    
    def test_anonymous_cannot_create(self):
        response = self.client.post('/api/articles/', {'title': 'New'})
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)
    
    def test_author_can_update_own(self):
        self.client.force_authenticate(user=self.author)
        response = self.client.patch(
            f'/api/articles/{self.article.pk}/',
            {'title': 'Updated'}
        )
        self.assertEqual(response.status_code, status.HTTP_200_OK)
    
    def test_other_user_cannot_update(self):
        self.client.force_authenticate(user=self.regular_user)
        response = self.client.patch(
            f'/api/articles/{self.article.pk}/',
            {'title': 'Hacked'}
        )
        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)
    
    def test_admin_can_delete(self):
        self.client.force_authenticate(user=self.admin)
        response = self.client.delete(f'/api/articles/{self.article.pk}/')
        self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)
```

---

## 12. สรุป Part 068

✅ **AllowAny** ทุกคนเข้าได้ (ไม่ต้อง login)
✅ **IsAuthenticated** ต้อง login เท่านั้น
✅ **IsAdminUser** ต้องเป็น staff/admin
✅ **IsAuthenticatedOrReadOnly** login ถึงแก้ไขได้ ไม่ login อ่านได้
✅ **Custom Permissions** extends `BasePermission` implements `has_permission()` และ `has_object_permission()`
✅ **Object-level permissions** ตรวจสอบสิทธิ์ต่อ object เช่น เป็น owner หรือไม่
✅ **OR, AND, NOT operators** รวม permissions หลายอัน (DRF 3.9+)
✅ **get_permissions()** กำหนด permissions ต่าง action ใน ViewSet
✅ **DjangoModelPermissions** ใช้ Django model permissions กับ DRF
✅ ทดสอบ permissions ด้วย `client.force_authenticate(user=...)`

## ➡️ ถัดไป: Part 069 - DRF Filtering and Pagination

*Part 068/100+ | Python Course - Beginner to World-Class*
