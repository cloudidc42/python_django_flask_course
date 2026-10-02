# Part 067: DRF Authentication

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Authentication ใน DRF
- ใช้ TokenAuthentication สำหรับ API
- ติดตั้งและใช้ JWT ด้วย djangorestframework-simplejwt
- เข้าใจ SessionAuthentication
- สร้าง Custom Authentication

---

## 1. Authentication ใน DRF

Authentication คือการตรวจสอบว่า "คุณเป็นใคร" ต่างจาก Permission ที่ตรวจว่า "คุณทำอะไรได้บ้าง"

DRF รองรับหลาย Authentication schemes:

| Scheme | วิธีส่ง | เหมาะสำหรับ |
|--------|---------|-------------|
| SessionAuthentication | Cookie | Web browser |
| BasicAuthentication | username:password ใน header | Testing |
| TokenAuthentication | Token ใน header | Mobile/SPA |
| JWT | JWT token | Modern API |

```python
# request.user และ request.auth
# - request.user: User instance หรือ AnonymousUser
# - request.auth: Token/JWT object (ขึ้นกับ scheme)
```

---

## 2. SessionAuthentication

ใช้สำหรับ web browser ที่ login ผ่าน Django session

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.BasicAuthentication',
    ],
}
```

```python
# urls.py - เพิ่ม login/logout สำหรับ Browsable API
urlpatterns = [
    path('api-auth/', include('rest_framework.urls')),  # login/logout
]
```

SessionAuthentication ทำงานร่วมกับ Django login อยู่แล้ว แต่ต้องมี CSRF token

---

## 3. TokenAuthentication

Token-based authentication: client รับ token แล้วส่งใน header ทุก request

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'rest_framework',
    'rest_framework.authtoken',  # ต้องเพิ่ม app นี้
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
        'rest_framework.authentication.SessionAuthentication',
    ],
}
```

```bash
python manage.py migrate  # สร้างตาราง Token
```

```python
# views.py - Login endpoint
from rest_framework.authtoken.models import Token
from rest_framework.response import Response
from rest_framework.views import APIView
from rest_framework import status
from django.contrib.auth import authenticate

class LoginAPIView(APIView):
    """Login และรับ Token"""
    permission_classes = []  # ไม่ต้อง authenticate ก่อน
    
    def post(self, request):
        username = request.data.get('username')
        password = request.data.get('password')
        
        if not username or not password:
            return Response(
                {'error': 'กรุณากรอก username และ password'},
                status=status.HTTP_400_BAD_REQUEST
            )
        
        # ตรวจสอบ credentials
        user = authenticate(request, username=username, password=password)
        
        if not user:
            return Response(
                {'error': 'Username หรือ password ไม่ถูกต้อง'},
                status=status.HTTP_401_UNAUTHORIZED
            )
        
        if not user.is_active:
            return Response(
                {'error': 'บัญชีนี้ถูกระงับ'},
                status=status.HTTP_401_UNAUTHORIZED
            )
        
        # สร้างหรือดึง Token
        token, created = Token.objects.get_or_create(user=user)
        
        return Response({
            'token': token.key,
            'user': {
                'id': user.id,
                'username': user.username,
                'email': user.email,
                'full_name': user.get_full_name(),
            }
        })


class LogoutAPIView(APIView):
    """Logout - ลบ Token"""
    
    def post(self, request):
        # ลบ token
        try:
            request.user.auth_token.delete()
        except Exception:
            pass
        return Response({'message': 'ออกจากระบบสำเร็จ'})
```

```python
# หรือใช้ built-in view
from rest_framework.authtoken import views as token_views

urlpatterns = [
    path('api/auth/token/', token_views.obtain_auth_token, name='obtain-token'),
]
```

### การใช้ Token

```bash
# Login รับ token
curl -X POST http://localhost:8000/api/auth/login/ \
  -H "Content-Type: application/json" \
  -d '{"username": "john", "password": "password123"}'

# Response:
# {"token": "9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b"}

# ส่ง token ใน header
curl -H "Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b" \
  http://localhost:8000/api/articles/
```

---

## 4. JWT Authentication ด้วย djangorestframework-simplejwt

JWT (JSON Web Token) มีข้อดีกว่า Token ธรรมดา:
- Stateless - ไม่ต้องเก็บใน database
- มี expiration
- มี refresh token
- ใส่ข้อมูลใน payload ได้

```bash
pip install djangorestframework-simplejwt
```

```python
# settings.py
from datetime import timedelta

INSTALLED_APPS = [
    # ...
    'rest_framework',
    'rest_framework_simplejwt',
    'rest_framework_simplejwt.token_blacklist',  # optional: blacklist tokens
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
        'rest_framework.authentication.SessionAuthentication',
    ],
}

# JWT settings
SIMPLE_JWT = {
    # ระยะเวลา access token มีอายุ
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=60),
    
    # ระยะเวลา refresh token มีอายุ
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    
    # Rotate refresh token (สร้าง refresh token ใหม่เมื่อ refresh)
    'ROTATE_REFRESH_TOKENS': True,
    
    # Blacklist refresh token เก่าหลัง rotate
    'BLACKLIST_AFTER_ROTATION': True,
    
    # อัปเดต last_login เมื่อ refresh
    'UPDATE_LAST_LOGIN': True,
    
    # Algorithm
    'ALGORITHM': 'HS256',
    
    # Header type
    'AUTH_HEADER_TYPES': ('Bearer',),
    'AUTH_HEADER_NAME': 'HTTP_AUTHORIZATION',
    
    # Token type
    'USER_ID_FIELD': 'id',
    'USER_ID_CLAIM': 'user_id',
    
    # Token classes
    'TOKEN_TYPE_CLAIM': 'token_type',
    'JTI_CLAIM': 'jti',
}
```

```python
# urls.py
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenVerifyView,
    TokenBlacklistView,
)

urlpatterns = [
    # Login - รับ access + refresh token
    path('api/auth/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    
    # Refresh - ใช้ refresh token รับ access token ใหม่
    path('api/auth/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    
    # Verify - ตรวจสอบว่า token valid
    path('api/auth/token/verify/', TokenVerifyView.as_view(), name='token_verify'),
    
    # Blacklist (logout)
    path('api/auth/token/blacklist/', TokenBlacklistView.as_view(), name='token_blacklist'),
]
```

### Custom JWT Claims

```python
# serializers.py
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer
from rest_framework_simplejwt.views import TokenObtainPairView

class CustomTokenObtainPairSerializer(TokenObtainPairSerializer):
    """เพิ่ม claims พิเศษใน JWT payload"""
    
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        
        # เพิ่มข้อมูลใน payload
        token['username'] = user.username
        token['email'] = user.email
        token['full_name'] = user.get_full_name()
        token['is_staff'] = user.is_staff
        
        # เพิ่ม roles/groups
        token['groups'] = list(user.groups.values_list('name', flat=True))
        
        return token
    
    def validate(self, attrs):
        data = super().validate(attrs)
        
        # เพิ่มข้อมูล user ใน response
        data['user'] = {
            'id': self.user.id,
            'username': self.user.username,
            'email': self.user.email,
            'full_name': self.user.get_full_name(),
        }
        
        return data


class CustomTokenObtainPairView(TokenObtainPairView):
    serializer_class = CustomTokenObtainPairSerializer


# urls.py
urlpatterns = [
    path('api/auth/token/', CustomTokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/auth/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
]
```

### การใช้ JWT

```bash
# Login
curl -X POST http://localhost:8000/api/auth/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "john", "password": "password123"}'

# Response:
# {
#   "access": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...",
#   "refresh": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9..."
# }

# ใช้ access token
curl -H "Authorization: Bearer eyJ0eXAiOiJKV1Qi..." \
  http://localhost:8000/api/articles/

# Refresh token
curl -X POST http://localhost:8000/api/auth/token/refresh/ \
  -H "Content-Type: application/json" \
  -d '{"refresh": "eyJ0eXAiOiJKV1Qi..."}'
```

### JavaScript/Axios ตัวอย่าง

```javascript
// JavaScript - จัดการ JWT
const API_URL = 'http://localhost:8000/api';

// Login
async function login(username, password) {
    const response = await fetch(`${API_URL}/auth/token/`, {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify({username, password})
    });
    
    const data = await response.json();
    
    if (response.ok) {
        // เก็บ tokens
        localStorage.setItem('access_token', data.access);
        localStorage.setItem('refresh_token', data.refresh);
    }
    
    return data;
}

// API request พร้อม JWT
async function apiRequest(url, options = {}) {
    const accessToken = localStorage.getItem('access_token');
    
    const response = await fetch(`${API_URL}${url}`, {
        ...options,
        headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${accessToken}`,
            ...options.headers
        }
    });
    
    // ถ้า 401 ให้ refresh token
    if (response.status === 401) {
        const refreshed = await refreshAccessToken();
        if (refreshed) {
            return apiRequest(url, options);  // retry
        }
        // logout
        localStorage.removeItem('access_token');
        localStorage.removeItem('refresh_token');
        window.location.href = '/login';
    }
    
    return response;
}

// Refresh access token
async function refreshAccessToken() {
    const refreshToken = localStorage.getItem('refresh_token');
    
    const response = await fetch(`${API_URL}/auth/token/refresh/`, {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify({refresh: refreshToken})
    });
    
    if (response.ok) {
        const data = await response.json();
        localStorage.setItem('access_token', data.access);
        return true;
    }
    return false;
}
```

---

## 5. Custom Authentication

```python
# authentication.py
from rest_framework.authentication import BaseAuthentication
from rest_framework.exceptions import AuthenticationFailed
from django.contrib.auth import get_user_model
import hashlib
import hmac
import time

User = get_user_model()

class APIKeyAuthentication(BaseAuthentication):
    """Authentication ด้วย API Key ใน header"""
    
    def authenticate(self, request):
        """
        ตรวจสอบ request
        Return: (user, auth) tuple หรือ None ถ้าไม่มี credentials
        Raise: AuthenticationFailed ถ้า credentials ผิด
        """
        api_key = request.META.get('HTTP_X_API_KEY')
        
        if not api_key:
            return None  # ไม่มี API key - ปล่อยให้ authentication อื่นจัดการ
        
        try:
            # ค้นหา user จาก API key
            from .models import APIKey
            api_key_obj = APIKey.objects.select_related('user').get(
                key=api_key,
                is_active=True
            )
            
            # ตรวจสอบ expiration
            if api_key_obj.expires_at and api_key_obj.expires_at < timezone.now():
                raise AuthenticationFailed('API Key หมดอายุแล้ว')
            
            # อัปเดต last_used
            api_key_obj.last_used = timezone.now()
            api_key_obj.save(update_fields=['last_used'])
            
            return (api_key_obj.user, api_key_obj)
        
        except APIKey.DoesNotExist:
            raise AuthenticationFailed('API Key ไม่ถูกต้อง')
    
    def authenticate_header(self, request):
        """สำหรับ WWW-Authenticate header เมื่อ 401"""
        return 'APIKey realm="api"'


class HMACAuthentication(BaseAuthentication):
    """Authentication ด้วย HMAC signature"""
    
    def authenticate(self, request):
        auth_header = request.META.get('HTTP_AUTHORIZATION', '')
        
        if not auth_header.startswith('HMAC '):
            return None
        
        try:
            _, credentials = auth_header.split(' ', 1)
            key_id, timestamp, signature = credentials.split(':')
        except ValueError:
            raise AuthenticationFailed('รูปแบบ Authorization header ไม่ถูกต้อง')
        
        # ตรวจสอบว่า timestamp ไม่เก่าเกิน 5 นาที
        try:
            ts = int(timestamp)
            if abs(time.time() - ts) > 300:
                raise AuthenticationFailed('Timestamp หมดอายุ')
        except ValueError:
            raise AuthenticationFailed('Timestamp ไม่ถูกต้อง')
        
        # ดึง user และ secret
        try:
            from .models import APICredential
            credential = APICredential.objects.get(key_id=key_id)
        except APICredential.DoesNotExist:
            raise AuthenticationFailed('Key ID ไม่ถูกต้อง')
        
        # ตรวจสอบ signature
        message = f'{request.method}\n{request.path}\n{timestamp}'
        expected = hmac.new(
            credential.secret.encode(),
            message.encode(),
            hashlib.sha256
        ).hexdigest()
        
        if not hmac.compare_digest(signature, expected):
            raise AuthenticationFailed('Signature ไม่ถูกต้อง')
        
        return (credential.user, credential)
```

---

## 6. กำหนด Authentication ต่อ View

```python
from rest_framework.authentication import SessionAuthentication
from rest_framework_simplejwt.authentication import JWTAuthentication

class PublicArticleView(generics.ListAPIView):
    """View ที่ไม่ต้อง authentication"""
    authentication_classes = []  # ไม่ต้อง authenticate
    permission_classes = []      # ไม่ต้อง permission
    queryset = Article.objects.filter(status='published')
    serializer_class = ArticleSerializer


class PrivateAPIView(generics.ListCreateAPIView):
    """View ที่รับเฉพาะ JWT"""
    authentication_classes = [JWTAuthentication]
    permission_classes = [IsAuthenticated]
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
```

---

## 7. สรุป Part 067

✅ **SessionAuthentication** ใช้กับ web browser ที่ login ผ่าน Django
✅ **TokenAuthentication** ส่ง token ใน `Authorization: Token <key>` header
✅ **JWT (simplejwt)** modern authentication พร้อม access/refresh tokens
✅ JWT มีข้อดีคือ stateless ไม่ต้องเก็บ session ใน database
✅ **Custom Authentication** สร้าง class ที่ extends `BaseAuthentication`
✅ **Custom JWT Claims** เพิ่มข้อมูลพิเศษใน token payload

## ➡️ ถัดไป: Part 068 - DRF Permissions

*Part 067/100+ | Python Course - Beginner to World-Class*
