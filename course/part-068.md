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

## 9. สรุป Part 068

✅ **AllowAny** ทุกคนเข้าได้ (ไม่ต้อง login)
✅ **IsAuthenticated** ต้อง login เท่านั้น
✅ **IsAdminUser** ต้องเป็น staff/admin
✅ **IsAuthenticatedOrReadOnly** login ถึงแก้ไขได้ ไม่ login อ่านได้
✅ **Custom Permissions** extends `BasePermission` implements `has_permission()` และ `has_object_permission()`
✅ **Object-level permissions** ตรวจสอบสิทธิ์ต่อ object เช่น เป็น owner หรือไม่
✅ **OR, AND, NOT operators** รวม permissions หลายอัน (DRF 3.9+)
✅ **get_permissions()** กำหนด permissions ต่าง action ใน ViewSet

## ➡️ ถัดไป: Part 069 - DRF Filtering and Pagination

*Part 068/100+ | Python Course - Beginner to World-Class*
